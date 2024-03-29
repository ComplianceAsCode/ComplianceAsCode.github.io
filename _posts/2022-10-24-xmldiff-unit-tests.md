---
layout: post
title: "Using xmldiff in Python unit tests"
categories: template
author: Jan Černý
author_url: https://github.com/jan-cerny
---

Recently, we have decided to improve the test coverage of the [ComplianceAsCode](https://github.com/ComplianceAsCode/content) build system by adding more unit tests for our Python modules.

Specifically, we have focused on testing code that works with XML.
We have been creating tests for methods that generate XML elements or generate XML trees or transform one XML tree to another.

At first sight, testing these types of methods looks easy.
We created some fixtures and then wrote some test cases with asserts counting the amount of generated elements and attributes and checking the expected values.
An example of this is below:

```python
def test_group_to_xml_element(group_selinux):
    group_el = group_selinux.to_xml_element()
    assert group_el is not None
    assert group_el.tag == {% raw %} "{%s}Group" % XCCDF12_NS {% endraw %}
    assert len(group_el.attrib) == 1
    assert group_el.get("id") == "xccdf_org.ssgproject.content_group_selinux"
    assert group_el.text is None
    ... snip ...
```

This is quite easy and most people would be fine with a test case like this.
The advantage of this approach was that every requirement on the tested method had its own assert so when test started to fail it was immediately obvious what is broken.
However, we didn't quite like it.
The expected XML structure generated by the tested method (`to_xml_element()` in the example above) isn't clear from the code.
The test can be quite long and it is laborious to write all the asserts for methods generating big XML trees with many child elements.
So we have started to look for options for improving the tests.

## Get familiar with xmldiff

We have discovered the [xmldiff](https://xmldiff.readthedocs.io/en/stable/) project.

It's a Python package that can be installed by `pip`:

```bash
$ sudo pip3 install xmldiff
```

It can be used both as a command line tool and a Python module.

Assuming that you have 2 XML files, `file1.xml` and `file2.xml`, run the following command:

```bash
$ xmldiff file1.xml file2.xml

[update-attribute, /ns0:Rule/ns0:platform[1], idref, "virtual"]
[update-text, /ns0:Rule/ns0:ident[1], "777777"]
```

The `xmldiff` command will return a list of actions.
This list of actions is so-called "Edit Script" and contains all changes needed to transform the first compared XML to the second compared XML.
In the example above, we can see there are two differences between the two XML files.
First is that the attribute `idref` on element described by XPath expression `/ns0:Rule/ns0:platform[1]` is changed to `virtual`.
Second is that the text of the element described by XPath expression `/ns0:Rule/ns0:ident[1]` is changed to `777777`.

In a Python script, you can call xmldiff this way:

```python
import xmldiff.main
diff = xmldiff.main.diff_files("file1.xml","file2.xml")
print(diff)
```

It seems that the `xmldiff` is very easy to use, so we have decided to use it in our unit tests.
The [xmldiff documentation](https://xmldiff.readthedocs.io/en/stable/) is a good starting point.

But, we have encountered some small caveats, which we will describe below.

## Passing XML trees to the library

Our methods usually return `xml.etree.ElementTree` instances, so we first used the `xmldiff.main.diff_trees()` method to compare them.
We put the expected output to a file in our test data directory and in the test we parsed the file and we put the parsed tree in a fixture.

The problem was that the xmldiff takes `lxml` instances and not `xml.etree` instances which we use, so we had to convert both of them to `lxml`.

This works quite fine.
In case of any random difference between the actual and the expected output the test would fail.
Our previous test then looked like this:

```python
def test_group_to_xml_element(group_selinux, group_selinux_xml):
    group_el = group_selinux.to_xml_element()
    group_tree = lxml.etree.fromstring(ET.tostring(group_el))
    diff = xmldiff.main.diff_trees(group_tree, group_selinux_xml)
    assert diff == []
```

## Handling white space

However, then we reviewed our code and we didn't like the saved XML test data &mdash; they were ugly, with no nice formatting.
So we decided to apply `xmllint` pretty format and then the XMLs look pretty.
But, the tests started to fail.

We have found that the `xmldiff` is very sensitive and produced a bunch of differences that we add newline and whitespace here and there.
We were wondering how to convince `xmldiff` to ignore the whitespace.
We didn't want to run `xmllint` command as a subprocess in our tests.
We tried to use [formatters](https://xmldiff.readthedocs.io/en/stable/api.html#using-formatters) but with no luck, xmllint still behaved sensitively to whitespace.
We were mainly concerned that the data in the stored form would be difficult to review and the whitespace sensitivity would make them cumbersome to maintain.
By accident, we have discovered that this behavior doesn't happen with the `xmllint.main.diff_files()` method.
That method isn't sensitive to whitespace or formatting of the XML files, so we can save them in a pretty format.
So we reworked our tests so that the test first saved the output of the tested method to a temporary file and then we called `xmllint.main.diff_files()` to compare this temporary file with our static file in test data.
The test function code is very easy and the test data can look pretty.
Moreover, we don't need to import `lxml`.

```python
def test_group_to_xml_element(group_selinux):
    group_el = group_selinux.to_xml_element()
    with temporary_filename() as real:
        ET.ElementTree(group_el).write(real)
        expected = os.path.join(DATADIR, "selinux.xml")
        diff = xmldiff.main.diff_files(real, expected)
        assert diff == []
```

Note: The `temporary_filename` is a context manager that gives us a temporary file name.

## Working with namespaces

One of our methods transforms a given XML tree to a different XML tree that differs in a couple of attributes and values but the rest of the tree is the same.
So we have compared the input of this method with the output of this method using `xmldiff` and we got the diff in the form of an Edit script.
Then, we had to solve how to write an assert that this Edit script is the expected one.
In other words, to verify that the `xmldiff` has given the expected diff.
We found that the items in the diff are Python `namedtuple`s and that we can easily create our own `namedtuple`s in the code and then check if they're present in the diff.

These tuples contain the description of the element using XPath. However, it uses the namespace prefix.
We were afraid that this prefix can become different easily.
Using the prefix without any mapping is not the way one normally works with namespaces.
But, there was no way to provide a correct XPath with the namespaces and the documentation doesn't mention how to do that.
So we have created a workaround that access the namespace map in the "new" `lxml` tree and we create a reverse mapping and then we save the actual prefix to a variable.

In the following example, we test that the 2 XML `lxml` trees differ in exactly one thing which is a value of the `id` attribute on the `definition` element, where the `definition` element belongs to the `"http://oval.mitre.org/XMLSchema/oval-definitions-5"` namespace.

```python
def test_foo(old, new):
    # create an inverted namespace map from new.nsmap
    # inverted map maps prefixes to namespace URIs
    inverted_new_nsmap = {v: k for k, v in new.nsmap.items()}
    # take the actually used prefix of the namespace
    prefix = inverted_new_nsmap["http://oval.mitre.org/XMLSchema/oval-definitions-5"]
    # perform the diff
    diff = set(xmldiff_main.diff_trees(old, new))
    # craft the expected value, use the prefix variable in the XPath expression
    action1 = xmldiff.actions.UpdateAttrib(
        node=f'/{prefix}:oval_definitions/{prefix}:definitions/{prefix}:definition[1]',
        name='id',
        value='oval:ssg-kerberos_disable_no_keytab:def:1')
    # assert that the expected value is in the diff
    assert action1 in diff
    # assert that no other value than the expected value is in the diff
    diff.remove(action)
    assert diff == set()
```

## Conditional imports

Another problem that we faced is that we wanted to use the `xmldiff` tests in our upstream and downstream CI.
Unfortunately, we discovered that the library isn't available as RPM, neither in Fedora nor in RHEL.
It's available only in PyPI.
That means we can't execute the tests in some of our test environments.
But, we wanted to still run the tests in the environments where `xmldiff` is available and at the same time not disable all the unit tests on the other systems. Fortunately, `pytest` has a very elegant method `importorskip()` that skips the test case when some module isn't available and still runs the other test cases.

We have used this method in every test function where we use `xmldiff`:

```python
def test_foo():
    …
    xmldiff_main = pytest.importorskip("xmldiff.main")
    diff = xmldiff_main.diff_files(real_file_path, expected_file_path)
    …
```

## Conclusion

The `xmldiff` library is very useful tool for comparing XMLs and writing unit tests for Python code working with XML.
We have successfully introduced multiple unit tests that leverage `xmldiff` in our project.
If you are curious about the full code, take a look for example to [test_build_yaml](https://github.com/ComplianceAsCode/content/blob/master/tests/unit/ssg-module/test_build_yaml.py).

However, for wider adoption in our project, we will need to make the `xmldiff` package present in Fedora and other Linux distributions.

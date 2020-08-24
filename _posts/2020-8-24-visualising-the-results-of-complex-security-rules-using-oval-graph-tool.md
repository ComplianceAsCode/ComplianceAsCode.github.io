---
layout: post
title: "Visualising the results of complex security rules using oval-graph tool"
categories: template
author: Jan Rodák
---

## Introduction

You scanned the computer with OpenSCAP and some rules in the security profile failed. Then you have a question: Why have the rules failed? The answer is simple. Take a look at the report and read what happened. But sometimes the report is not easy to read. Some rules are very complex and you can easily get lost in the information in the report.

See, for example, this screenshot of a HTML report showing results of a rule:

![Rule details](/assets/images/rule_details.png)

As you can see in the screenshot, HTML reports contain a lot of information, but when complex rules fail unexpectedly, it is hard to find out the real reason for the failure.

## Problem: Best is not good enough

So far, the HTML report has been the best option for visualising OVAL results. But, it has some problems. Rules consist of OVAL tests that have a tree-like logical relationship. It is not necessarily a binary tree. Negation also works in this tree. Unfortunately, the information about the relationship between the tests is missing in the HTML report.  And this information about negation and relationship can help users.

Let's show the problem that can occur in the case of a complex rule. The picture shows the result of the rule´s details, where the results of individual OVAL tests are listed. The user would appreciate if the display of results also contained a logical connection between the individual tests.

![OVAL results](/assets/images/oval_results.png)

In this case, only information about individual tests are displayed. Take a look at the second test, which results in TRUE, and the other four tests below that TRUE test. These tests are linked by the OR operator, which is negated, and thus change the entire result of the rule. You can’t get this information from the HTML report. You can only get this information from another possible type of SCAP scanner report, so called ARF file, which is an XML file containing all the information about the performed scan.

## Solution

This problem is solved by a tool called  [oval-graph](https://github.com/OpenSCAP/OVAL-visualization-as-graph). This tool is easy to install: It is available on PyPi or as a Fedora RPM. It is easy to use. This tool can display the complete result of the rule. In the form of a simple tree. That allows you to understand the complex rules in the blink of an eye.
See, for example, the output of tool:

![OVAL graph](/assets/images/tree.png)

This tool shows the relation between tests, negation of operator and negation of test. See examples:

![negation](/assets/images/negation.png)

Generated graph is interactive. Click on test for details of test.

![test details](/assets/images/open_node.gif)

Also you can click on the operator and hide tests, definitions and criteria related to this operator.

![hide nodes](/assets/images/hide_nodes.gif)

This tool has a cool feature. Output of the tool can be an HTML or JSON file from which users can restore the rule graph. In the JSON file, the user can easily change the test result and then create a new recalculated graph from the modified JSON. So try to solve your problem in rough.

## Installation

### Fedora (EPEL)

```bash
dnf instal oval-graph
```

### Any OS - use PyPi

```bash
pip3 install oval-graph
```

## Everyday usage

With command arf-to-graph you can easily transform ARF xml reports to nice graphs. This command consumes the rule name or regular expression of rule name and the ARF output, which is one of possible standardized formats for results of SCAP-compliant scanners.

For example: You use OpenSCAP to scan your system. The command below scans the computer with the Fedora operating system and uses the OSPP profile. This command generates HTML report (report.html) and XML report (arf.xml).

```bash
oscap xccdf eval \
 --profile ccdf_org.ssgproject.content_profile_ospp \
 --results-arf arf.xml \
 --report report.html \
 /usr/share/xml/scap/ssg/content/ssg-fedora-ds.xml
```

The output of oscap command looks like this:

![oscap output](/assets/images/oscap_output.png)

In the output of the command we can see a failed rule named “Enable FIPS Mode” (xccdf_org.ssgproject.content_rule_enable_fips_mode). We can read some information about this rule  in the HTML report.

See information in HTML:

![fips rule](/assets/images/fips.gif)

This rule looks like one of complex rules. There  are many tests and we can't see any relation between tests. Fortunately, this relation can be displayed with command arf-to-graph.

Here is the command. First parameter is the arf file and second is part of the rule name.

```bash
arf-to-graph arf.xml enable_fips_mode
```

This command generates a graph and saves a file named ```graph-of-<rule_id>-<date>.html``` (the date the graph was created) in the working directory. Then, it opens the generated file in your web browser. Learn more information about usage in the user [Guide](https://github.com/OpenSCAP/OVAL-visualization-as-graph/blob/master/docs/GUIDE.md).

You can read more about generating ARF report files using OpenSCAP in the OpenSCAP User [Manual](https://github.com/OpenSCAP/openscap/blob/maint-1.3/docs/manual/manual.adoc). Or you can use test arf files from [repository](https://github.com/OpenSCAP/OVAL-visualization-as-graph) /tests/test_data.

Here is a live demo of the generated file. This graph is interactive. You can hide part of the rule or display any information about the test in the rule. In the graph we can see in the blink of an eye where the problem is.

***html of fips rule***

## Conclusion

The oval-graph tool helps  users solve the complex rule conditions that are contained in SCAP security profiles. This tool contains commands which users can use to solve their problems. In the future, we plan to put the generated graphs  to the HTML report to make the ultimate report which users appreciate. I believe it will be possible to implement this feature directly in the OpenSCAP scanner.

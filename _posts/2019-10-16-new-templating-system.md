---
layout: post
title: "Simplifying templates in ComplianceAsCode"
categories: template
author: Jan Černý
---
In ComplianceAsCode content project, there are many similar rules. But we don’t like to duplicate code and we discourage copy-pasting. Instead, we prefer to generate content (e.g: OVAL, Ansible, Bash, etc.) using templates. Lately, we have changed the way these templates work. This change made using templates easier for the content authors.

In this article, we will describe the reasons for this change and we will show how to use the new templating system.

### Why we made this change

The old system was working, and it was definitely a big improvement over copy-pasting the code. But it had multiple problems.

First problem was that the `CSV` files which contained parameters for the templates were located at other directories than the `rule.yml` files. This wasn’t convenient because contributors had to browse the directory tree to find the right `CSV` file. Moreover, there was a different set of files in each product (e.g: RHEL8, RHEL7, Fedora, etc.). And there were many `CSV` files in shared directory as well, shared within multiple products. For example, there was a file with data for a template which generates content for mount options checks in the `shared` directory, in the RHEL 7 product directory, and in the RHEL 8 product directory as well:
 - [Shared mount_options.csv](https://github.com/ComplianceAsCode/content/blob/54aa23363d7569b3f8f432d1732d2cb68cf5e5a0/shared/templates/csv/mount_options.csv)
 - [RHEL8 mount_options.csv](https://github.com/ComplianceAsCode/content/blob/54aa23363d7569b3f8f432d1732d2cb68cf5e5a0/rhel8/templates/csv/mount_options.csv)
 - [RHEL7 mount_options.csv](https://github.com/ComplianceAsCode/content/blob/54aa23363d7569b3f8f432d1732d2cb68cf5e5a0/rhel7/templates/csv/mount_options.csv)

These multiple `CSV` files mostly weren’t kept consistent. The fact that some data are specific for a given product also complicated the creation of a new product content because new CSVs had to be created again.

There wasn’t any visible connection between the rules and the templates. When there was no `oval` or `ansible` subdirectories in the rule directory, it could mean that the rule is templated, but that could also mean that no content for that rule exists complicating pull request reviews. The build system generated content from `CSV` files with no respect to rules. The generated content was mapped to rules later in the build process based on IDs. The CSVs contained many lines that weren’t related to any rule in the currently built product, which means that a lot of content was generated and then dropped later in the build process.

From another point of view, there was also no connection from the `CSV` files to the rules. When we took a look at a line in a `CSV` file it wasn’t obvious to which rule that line was related. We could guess, but to verify you needed a deep understanding of the build system code.

Also, there was a big portion of complex code that populated the templates and processed the template data. There was a special `Python` class for each `CSV` file and the for each of them the behavior was different. Some classes generated content for multiple rules from a single CSV entry at once. For example, from each line in `CSV` `file_dir_permissions.csv` up to four OVAL files could be generated, each of them for a different `XCCDF` rule, but one of them was never used.

Introduction of templates was considered a great improvement at that time but its implementation had a lot of quirks. We wanted to provide something more straightforward and easier to use.

### New approach

The main change is that the templates are now rule-centric, and are integrated into the YAML rule file. To use a template we can specify the `template` key in the `rule.yml` file when creating a new rule. We simply provide the name of template and the parameters directly under `rule.yml` and that's it - we don’t need to edit any others files.

Making templates dependent on rules also means that we can easily check if a rule generates templated content just by checking if the `rule.yml` file contains a `template` key. In the past we had to examine all the `CSV` files.

We made the templating logic very simple: Each template has its name and a single implementation in each of the supported languages. The template variables are passed directly to `Jinja 2` processing engine, without using custom `Python` classes. There are callback functions in [`ssg/templates.py`](https://github.com/ComplianceAsCode/content/blob/master/ssg/templates.py) that can process or check the data. The generated files have the same name as the rule ID, they differ only in the extension and target path. Even if this might seem expected, it wasn’t like that in the old implementation where each template had a specific behavior.

### Practical example

In this example, we will use the new templating system in rule “Enable the OpenSSH service”. The rule metadata (YAML file) is located in `linux_os/guide/services/ssh/service_sshd_enabled/rule.yml`.

To use a template, we add the `template` key in the `rule.yml` file. It has two mandatory subkeys: First is `name`, which specifies the name of the template. Second key is `vars` which will contain pairs of template parameters and their values. The list of available templates and their parameters can be found in the [Developer guide](https://github.com/ComplianceAsCode/content/blob/master/docs/manual/developer_guide.adoc#73-templating).

The template that generates content that checks enablement of system services is called `service_enabled` and it has two parameters: `servicename`, which should contain the name of the service and `packagename` which should contain the name of software package (`RPM` or `DEB`) which provides this service.
```
template:
    name: service_enabled
    vars:
        servicename: sshd
        packagename: openssh-server
```
If you build the content, this entry will cause that OVAL definition and Ansible and Bash remediations for this rule are generated and included in the built datastream. You can check that by looking into the datastream, or you can check `build/<product>/checks/oval` and `build/<product>/fixes_from_templates/<language>` directories.

The content will be generated for all the platforms that the rule is applicable to according to `prodtype` key defined under the `rule.yml` file. That’s usually what we want. But, in our examplary rule, there is a problem on OpenSUSE systems that the package which provides the `sshd` service isn’t called `openssh-server` like on Red Hat Enterprise Linux distributions, but only `openssh`. To address this problem, we can use parameterized keys, which is a similar mechanism that we have been using for identifiers and references. We can append `@` followed by a product ID to the key. This allows us to set a specific value for OpenSUSE but the default value will be used for all other remaining products.
```
template:
    name: service_enabled
    vars:
        servicename: sshd
        packagename: openssh-server
        packagename@opensuse: openssh
```

By default, we generate every type of content that we have a template for - if the template contains OVAL, Ansible, Puppet and Bash templates, all four types of content will be generated. This is again what we usually want. If we want to disable generation of Ansible content for this rule it’s possible. The `template` key has also a third, optional key `backends`, that allows to modify this default behavior. To disable Ansible content for this rule, we will add the `backends` section with `ansible: "off"`.
```
template:
    name: service_enabled
    vars:
        servicename: sshd
        packagename: openssh-server
        packagename@opensuse: openssh
    backends:
        ansible: "off"
```
Note: The quotes around `"off"` are mandatory.

### Migration of custom content

If you have a downstream patch that adds content on the top of upstream source code, some action may be required from you. To migrate the data from `CSV` files to `rule.yml` files you can use our python script, which we also used to migrate the data. This script is located under [`utils/migrate_template_csv_to_rule.py`](https://github.com/ComplianceAsCode/content/blob/master/utils/migrate_template_csv_to_rule.py). It populates all the `rule.yml` files with the data from the `CSV` files.

However, some problems might occur and they need to be fixed manually. The most common issue is that in the old system you could have templated OVALs generated from CSVs which don’t have a corresponding rule, but they are referred by a static OVAL in some other rule. Unfortunately, this isn’t possible anymore because the new system is based purely on rules. We recommend creating new rules that contain a template key to generate these OVALs. Another option is to change the affected OVAL content to use CPE applicability or Jinja macros instead of `extend_definition` elements. The dependent checks can be also generated and put as static files to `shared/checks`.

If you have a custom template in your downstream patch, you need to add it to `ssg/templates.py` and provide a data processing callback. More information about that can be found in the upstream [Developer guide](https://github.com/ComplianceAsCode/content/blob/master/docs/manual/developer_guide.adoc#73-templating).

### Conclusion
We hope that with these changes described above using templates in our project will be now easier to use and also will allow us to further expand the templates. Moreover, the [Developer guide](https://github.com/ComplianceAsCode/content/blob/master/docs/manual/developer_guide.adoc#73-templating) now covers the new templating system in detail. 

In future we will need more templates to consolidate other rules. Also, the template system is currently missing input sanitizing and type checking before the data from `rule.yml` keys are substituted into the content.

---
layout: post
title: "Simplifying templates in ComplianceAsCode"
categories: template
---
In ComplianceAsCode content there are many similar rules. But we don’t like duplicate code and we discourage copy-pasting. Instead, we prefer to generate OVAL checks, Ansible and Bash code using templates. Lately, we have changed the way these templates work. This change makes using the templates easier for the content authors.

In this article, we will describe the reasons for this change and we will show using the new templating system in practice.

### Why we made this change

The old system was working, and it definitely was a big improvement over copy-pasting the code.  But it had multiple problems.

First problem was that the CSV files which contained parameters for the templates were located at other directories than the rule.yml files. This wasn’t convenient because contributors have to  browse the directory tree to find the right CSV file. Moreover, there was a different set of files in each product (rhel6, rhel7, fedora, etc.)  And there were many CSV files in shared directory as well, shared within multiple products. For example there is a file with data for template which generates content check in for mount options in shared directory, in RHEL 7 product directory and in RHEL 8 product directory as well: 
 - [Example 1](https://github.com/ComplianceAsCode/content/blob/master/shared/templates/csv/mount_options.csv)
 - [Example 2](https://github.com/ComplianceAsCode/content/blob/master/rhel8/templates/csv/mount_options.csv)
 - [Example 3](https://github.com/ComplianceAsCode/content/blob/master/rhel7/templates/csv/mount_options.csv)

The multiple CSV files mostly weren’t kept consistent. The fact that some data are specific for product also complicated the creation of a new product content because new CSVs have to be created again.

There wasn’t any visible connection between the rules and the templates. When there was no oval or ansible subdirectories in rule directory, it could mean that the rule is templated, but that could also mean that no content for that rule exists. That sometimes complicated the pull request reviews. The build system generated content from CSVs with no respect to rules. The generated content was mapped to rules later in the build process based on IDs. The CSVs contained many lines that weren’t related to any rule in the currently built product, which means that a lot of content was generated and then dropped later in the build process.

From another point of view, there was also no connection from the CSVs to the rules. When we took a look at a line in CSV it wasn’t obvious to which rule is that line related. We could guess, but to verify you need a deep understanding of the build system code.

Also, there was a big portion of complex code that populated the templates and processed the template data. There was a special Python class for each CSV file and behavior of each of them was different. Some classes generated content for multiple rules from a single CSV entry at once. For example, from each line in CSV file_dir_permissions.csv up to 4 OVAL files could be generated, each of them for a different XCCDF rule, but one of them was never used.

Even if introducing the templates in the past was a great improvement its implementation had a lot of quirks. We wanted to provide something more straightforward and easier to use.

### New approach

The main change is that the templates are now based on rules and are integrated in YAML rule file. To use a template we can specify “template” key in the rule.yml when creating a new rule. We simply provide the name of template and the parameters directly in rule.yml. And we don’t need to edit any other file than rule.yml. 

Making templates dependent on rules also means that we can easily check if a rule generates templated content just by checking if the rule.yml contains “template” key. In the past we had to examine all the CSV files.

We made the templating logic very simple: Each template has its name and a single implementation in each of supported languages. The template variables are passed directly to Jinja 2 processing engine, without using custom Python classes. There are callback functions in ssg/templates.py that can process or check the data. The generated files have the same names as the rule ID, they differ only in the extension and target path. Even if this might seem expected, it wasn’t like that in the old implementation where each template had a specific behavior.

### Practical example

In this example, we will use the new templating system in rule “Enable the OpenSSH service”. The rule metadata (YAML file) is located in `linux_os/guide/services/ssh/service_sshd_enabled/rule.yml`.

To use a template, we add “template” key in rule.yml. It has two mandatory subkeys: First is “name”, which specifies the name of the template. Second key is “vars” which will contain pairs of template parameters and their values. The list of available templates and their parameters can be found in the Developer guide.

The template that generates content that checks enablement of system services is called service_enabled and it has 2 parameters: “servicename”, which should contain the name of the service and “packagename” which should contain the name of software package (RPM or DEB) which provides this service.
```
template:
    name: service_enabled
    vars:
        servicename: sshd
        packagename: openssh-server
```
If you build the content, this entry will cause that  OVAL definition and  Ansible and Bash remediations for this rule are generated and included in the built datastream. You can check that by looking into the datastream, or you can check build/<product>/checks/oval and build/<product>/fixes_from_templates/<language> directories.

The content will be generated for all the platforms that the rule is applicable to according to “prodtype” key defined in rule.yml. That’s usually what we want. But, in our example rule, there is a problem on OpenSUSE systems the package that provides sshd service isn’t called “openssh-server” like on Red Hat distributions, but only “openssh”.  To address this problem, we can use parameterized keys, which is a similar mechanism that we have been using for identifiers and references. We can append @ followed by a product ID to the key. This allows us to set a specific value for OpenSUSE but the default value will be used in all remaining products.
```
template:
    name: service_enabled
    vars:
        servicename: sshd
        packagename: openssh-server
        packagename@opensuse: openssh
```

By default, we generate every type of content that we have a template for - if the template contains OVAL, Ansible, Puppet and Bash templates, all 4 types of content will be generated. This is again what we usually want.  If we want to disable generation of Ansible content for this rule it’s possible. The “template” key has also a third, optional key “backends”, that allows to modify this default behavior. To disable Ansible content for this rule, we will add the “backends” section with “ansible: “off””. 
```
template:
    name: service_enabled
    vars:
        servicename: sshd
        packagename: openssh-server
        packagename@opensuse: openssh
    backends:
        ansible: “off”
```
Unfortunately, the quotes around “off“ are mandatory.

### Migration of custom content

If you have a downstream patch that adds content on the top of upstream, some action will be needed from you. To migrate the data from CSVs to rule.yml files you can use our python script, which we also used to migrate the data in upstream. This script is located in utils/migrate_template_csv_to_rule.py. This will populate all the rule.yml files with the data. 

However, some problems might occur and they need to be fixed manually. The most common issue is that in the old system you could have templated OVALs generated from CSVs which don’t have a corresponding rule, but they are referred by a static OVAL in some other rule. Unfortunately, this isn’t possible anymore because the new system is based purely on rules. We recommend creating new rules that contain a template key to generate these OVALs. Another option is to change the affected OVAL content to use CPE applicability or Jinja macros instead of extend_definition. The dependent checks can be also generated and put as static files to shared/checks.

If you have a custom template in your downstream patch, you need to add it to “ssg/templates.py” and provide a data processing callback. More information about that can be found in the upstream Developer Guide.

### Conclusion
We hope that with the changes described above using templates in our project will be now easier to use and also will allow us to further expand the templates. Moreover, the Developer guide now covers the new templating system in detail. 

In future we will need more templates to consolidate other rules. Also, the template system is currently missing input sanitizing and type checking before the data from rule.yml keys are substituted into the content.

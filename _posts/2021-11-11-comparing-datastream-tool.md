---
layout: post
title: "Turning the Comparing Datastream Tool into a Service"
categories: template
author: Gabriel Becker
---

## Comparing Datastream Tool

Datastream files are usually big XML files that are meant to be processed by machines. When humans have to take a peek into it and see what's going on, it can get frustrating really fast. The ComplianceAsCode/content project aims for more abstracted ways of dealing with SCAP standard which helps developers creating content. But in the end, datastream files are generated and one change made to a particular file may be only reflected in this final artifact. If you want to make sure that your change was actually propagated in the right way, you may have to look at those big XML files.

The compare_ds.py is a tool written in Python that is able to compare two datastream files in a smart way partially solving this problem. The tool is able to parse datastream files and extract relevant information for the end user. This information can then be compared with a different version of the datastream providing informatino on what exactly has changed between these two versions.

### How to Run the Comparing Tool

First you have to generate two datastream files. You can generate them using the usual `./build_product <product>` command, then copy the datastream to a different folder (since `./build_product` removes everything from `build` directory every time it gets executed). When you have both artifacts in place you can run (from project's root directory):

`$ utils/compare_ds.py <old_datastream> <new_datastream>`

A change to a template's attribute value in the rule [`sshd_enable_strictmodes`](https://github.com/ComplianceAsCode/content/blob/master/linux_os/guide/services/ssh/ssh_server/sshd_enable_strictmodes/rule.yml):

```diff
diff --git a/linux_os/guide/services/ssh/ssh_server/sshd_enable_strictmodes/rule.yml b/linux_os/guide/services/ssh/ssh_server/sshd_enable_strictmodes/rule.yml
index 31968bc560a..b22e2046772 100644
--- a/linux_os/guide/services/ssh/ssh_server/sshd_enable_strictmodes/rule.yml
+++ b/linux_os/guide/services/ssh/ssh_server/sshd_enable_strictmodes/rule.yml
@@ -56,4 +56,4 @@ template:
         missing_parameter_pass: 'true'
         parameter: StrictModes
         rule_id: sshd_enable_strictmodes
-        value: 'yes'
+        value: 'no'
```

Would produce a similar diff output:

```diff
Rule 'xccdf_org.ssgproject.content_rule_security_patches_up_to_date' points to 'security-data-oval-com.redhat.rhsa-RHEL8.xml' which isn't a part of the old datastream
bash remediation for rule 'xccdf_org.ssgproject.content_rule_sshd_enable_strictmodes' differs:
--- old datastream
+++ new datastream
@@ -13,10 +13,10 @@
 if [ -z "$line_number" ]; then
 # There was no match of '^Match', insert at
 # the end of the file.
- printf '%s\n' "StrictModes yes" >> "/etc/ssh/sshd_config"
+ printf '%s\n' "StrictModes no" >> "/etc/ssh/sshd_config"
 else
 head -n "$(( line_number - 1 ))" "/etc/ssh/sshd_config.bak" > "/etc/ssh/sshd_config"
- printf '%s\n' "StrictModes yes" >> "/etc/ssh/sshd_config"
+ printf '%s\n' "StrictModes no" >> "/etc/ssh/sshd_config"
 tail -n "+$(( line_number ))" "/etc/ssh/sshd_config.bak" >> "/etc/ssh/sshd_config"
 fi
 # Clean up after ourselves.

ansible remediation for rule 'xccdf_org.ssgproject.content_rule_sshd_enable_strictmodes' differs:
--- old datastream
+++ new datastream
@@ -24,7 +24,7 @@
 path: /etc/ssh/sshd_config
 create: true
 regexp: (?i)^\s*StrictModes\s+
- line: StrictModes yes
+ line: StrictModes no
 state: present
 insertbefore: ^[#\s]*Match
 validate: /usr/sbin/sshd -t -f %s
```

These are the Ansible and Bash snippets present in the datastream files. As you can see, a simple
change can propagate to many different places. OVAL checks were also changed between these datastream files
but the tool does not support this kind of comparison at this moment.

One of the main ideas of this tool is to answer the question:

  * Am I making any undesired changes to the content?

In such a complex project that involves many aspects, any tool that supports identifying potential problems is
worth the investment.

## Integrating with Github Actions

While this tool can be used locally for comparing two different built datastreams, its full potential 
is actually turning it into a service that runs on every Pull Request proposed to the project. This way,
Pull Request reviewers can easily review differences and act upon.

Based on this premise, we have introduced a new Github Action Job that runs on a Pull Request, generates two datastream files,
one from current main branch and one containing the changes proposed by the Pull Request. With these two files, it runs the
`compare_ds.py` tool producing an output that contains the differences between them. The job also takes advantage of [Content Test Filtering](https://github.com/mildas/content-test-filtering) tool to detect which product it should build based on what are the changes in the Pull Request similarly as it's described in [SSGTS GH Actions](https://complianceascode.github.io/template/2021/08/27/integrating-ssgts-into-gha.html).

### Automatically Posting a Comment on a Pull Request

Sometimes it can get complicated to find log of test results in Github Actions, so the tool's output is posted as a comment in the original Pull Request and can be reviewed by anyone. This greatly increases the visibility of the service.

The following picture is an example of a comment:

![Compare DS Output Example](/assets/images/compare_ds_example.png)

Note: Every time the Pull Request is updated, the comment is replaced with the newest output.

## Limitations and Future Work

There are some limitations on what the `compare_ds.py` can do, OVAL checks are not fully supported and
not all remediation types are supported. Recently the tool has been expanded to detect differences in
CPE changes, for example if a pull request restricts a rule to run only on containers, this change would
be highlighted by the tool.

Future updates to the tool should automatically reflect in the Github Actions Job, so if someone implement new
features they will appear in the comments posted by the service.

This tool has great potential to help reviewers to speed up the review process and can also be used by developers
to spot any mistake or undesired change to the content.

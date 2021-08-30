---
layout: post
title: "Integrating The SSG Test Suite into GitHub Actions"
categories: template
author: Gabriel Becker
---

### Introduction

As described in [Testing Rules and Remediations]({% post_url 2021-03-25-tests_howto %}), the SSG Test Suite (SSGTS) is a tool that
is able to verify that a check passes or fails as expected on a given system. It was initially created to be run locally but as CI systems got more and more sophisticated, it become natural to integrate the Test Suite into GitHub Actions as it provides almost unlimited resources for public organizations for free.


### Github Actions

Github Actions is a platform that provides Continuous Integration/Continuous Development capabilities. You can automate your builds, tests, etc. It also provide a set of systems you can use with various OSes supported. Containers are also supported. For more details about Github Actions please refer to the [official documentation](https://github.com/features/actions).

### Content Test Filtering

[Content Test Filtering](https://github.com/mildas/content-test-filtering) (CTF) is a tool that identifies changes proposed in a pull request and detects which rules are affected by these changes.
This information allows to decide which rules should be tested by the automation. The tool also provides a human readable output that can be used to run tests locally. But this post approaches the machine readable format that is used the automation workflow.

### Integration

In Github Actions, there is a concept called `workflow`. It defines a set of instructions that will be executed whenever an event happen, for example a new pull request is opened. For the integration, a new `workflow` [file](https://github.com/ComplianceAsCode/content/blob/master/.github/workflows/ssgts.yaml) has been created to integrate the SSGTS and CTF in order to enable tests to be automatically executed by Github Actions.

This `workflow` defines the following actions:

- Trigger CTF on pull request changes to detect if any rule should be tested
- Build the content required for testing the rules (the product to be built is detected by CTF)
  - This content is currently built on a Fedora container as it contains the latest OpenSCAP package available
- Trigger SSGTS using rules provided by CTF
  - Tow potential instances of SSGTS are executed, depending which of the remediation are needed to be executed. This is also decided by CTF.
- Store log artifacts if tests fail.
  - These artifacts can be downloaded and inspected locally.

Let's take a look at this exemplary [pull request](https://github.com/ComplianceAsCode/content/pull/7478):

I edited the file [sshd_disable_compression/oval/shared.xml](https://github.com/ComplianceAsCode/content/blob/master/linux_os/guide/services/ssh/ssh_server/sshd_disable_compression/oval/shared.xml) and modified one of the regexes used for the OVAL check for demonstration purposes.

![OVAL Check Modification](/assets/images/ssgts_gha/oval_check_modified.png)

This modification will make the CTF identify that this rule needs to be tested, because it's check has changed and tests must be executed in order to confirm that no regressions were introduced.

The details of the `workflow` can be found in the checks section in the Pull Request's page or in the [checks tab](https://github.com/ComplianceAsCode/content/runs/3443824562?check_suite_focus=true):

![Check Status](/assets/images/ssgts_gha/check_status.png)

When you go into details, you see the following task, it contains the output of CTF:

![CTF Output](/assets/images/ssgts_gha/ctf_output.png)

In this case it has detected that the rule `sshd_disable_compression` needs to be tested using the product `rhel8`. Both remediation types also need to be tested because it's a change to the OVAL check, so both remediations should be run to verify that they are still aligned with the check.

Note: The `sshd_disable_compression` is applicable to many products, but CTF usually gives `rhel` variants precedence. If your PR targets a rule that is exclusive to a single product, it will report this product. Improvements to CTF are planned and how it reports data is a living topic.

Then, further down in the `workflow` you can find tests being run:

![Run Tests and Remediate with Bash](/assets/images/ssgts_gha/tests_bash.png)

If any of the tests would fail, log artifacts can be downloaded and investigated locally, for example:

![Logs](/assets/images/ssgts_gha/logs.png)

### Known Issues

The Content Test Filtering tool currently has some limitations. For example, it doesn't report a brand new rule introduced by the PR, nor when a test scenario is modified. These issues are being reported in the [issues page](https://github.com/mildas/content-test-filtering/issues) and will be eventually fixed.

Rules that require services do not play well with container so it will always fail. The solution is to use virtual machines, but this means supporting our own worker instances in Github Actions and this is not planned at the moment.

### Future Steps

- Make sure that Content Test Filtering reports correct and useful information
- Add support to more OSes where the tests are run.
  - Currently it runs only on a Fedora container and it can differ from other OSes in many aspects, for example package names, package manager, etc.

### Conclusion

The integration of SSGTS with a Continuous Integration system (e.g. Github Actions) had been a long time desire of the ComplianceAsCode/content contributors, it helps reducing time during reviews because there are results from test scenarios right away removing the need to run tests locally. Sometimes we cannot escape from running tests locally due to some constraints, but there are many cases where it plays well and it's definitely worth to keep this integration working.


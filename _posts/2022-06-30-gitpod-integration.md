---
layout: post
title: "Integration of Gitpod.io with ComplianceAsCode/content project"
categories: template
author: Gabriel Becker
author_url: https://github.com/ggbecker
---

## Introduction

Now you can take advantage of [`gitpod.io`](https://www.gitpod.io/) to create fresh development environments of [ComplianceAsCode/content](https://github.com/ComplianceAsCode/content) project.
You just need a github account to start benefiting from this integration. Everything will run on your web browser and no additional software is needed to be installed on
your system.

### Where It Can Be Used

You can use the tool to contribute to the project, to review pull requests or pretty much anything if you want to contribute to the project.

For example, the following link will create a ready-to-code development environment with all the required dependencies installed and a fully
working test environment (The default option for the testing environment is Fedora, but you can use other distributions):

[https://gitpod.io/#https://github.com/ComplianceAsCode/content](https://gitpod.io/#https://github.com/ComplianceAsCode/content)


![Visual Studio Code running on a Gitpod Environment](/assets/img/gitpod_env.png)
\
While reviewing pull requests in the project, a comment with a button will be displayed and it will
create an environment that contains the development environment plus the code of the particular
pull request. For example, see this [comment](https://github.com/ComplianceAsCode/content/pull/9072#issuecomment-1171045995):


![Comment on a Pull Request containing links that create Gitpod Environments with different parameters](/assets/img/gitpod_pr_comment.png)


## Testing Environment

A fresh testing environment should be ready to be used after the environment is deployed.

Run configurations are also available, which means you can simply activate the Run configurations as you would normally do in your
Visual Studio Code instance.

At this moment, only Fedora container images are available. The intent is to expand the scope to other distributions.
If you are insterested in contributing new images, please do so by submitting your Dockerfile here: https://github.com/ComplianceAsCode/content/tree/master/Dockerfiles

These images can be used in the `gitpod.io` integration to generate an environment specifically for you distribution.

The following parameters are available in order to customize the deployment:


* `PRODUCT`: The product to be built (`./build_product $PRODUCT`). Default is `fedora`.

* `CONTAINER`: The Dockerfile to be used when creating the testing container environment(it will look for a file like: `Dockerfiles/test_suite-$CONTAINER`). This defaults to `PRODUCT` if not set.

* `CONTAINER_VERSION`: Optional. If the Dockerfile supports defining the distribution version the same as defined in [test_suite-fedora](https://github.com/ComplianceAsCode/content/blob/bf5f0381df44a2079ec356d683c091b1b828931f/Dockerfiles/test_suite-fedora#L2). Example: `fedora:34`, `fedora:latest`

* `CPE`: Optional. Inject the CPE identifier into the testing environment so it makes the content applicable in case the product and container are different distributions. You can run the following command to obtain this information (in the testing target environment):
    * `cat /etc/os-release | grep CPE_NAME`

Note: All the parameters need to be [URL encoded](https://www.w3schools.com/tags/ref_urlencode.asp).

See below examples of how to invoke `gitpod.io` with parameters:


Product RHEL8 with a Fedora 34 container testing environment ([Direct link](https://gitpod.io/#PRODUCT=rhel8,CONTAINER=fedora,CONTAINER_VERSION=fedora%3A34,CPE=cpe%3A%2Fo%3Afedoraproject%3Afedora%3A34/https://github.com/ComplianceAsCode/content)):


```
https://gitpod.io/#PRODUCT=rhel8,CONTAINER=fedora,CONTAINER_VERSION=fedora%3A34,CPE=cpe%3A%2Fo%3Afedoraproject%3Afedora%3A34/https://github.com/ComplianceAsCode/content
```

Product Oracle Linux 8 with a Oracle Linux 8 container testing environment ([Direct link](https://gitpod.io/#PRODUCT=ol8,CPE=cpe%3A%2Fo%3Aoracle%3Alinux%3A8/https://github.com/ComplianceAsCode/content)):
```
https://gitpod.io/#PRODUCT=ol8,CPE=cpe%3A%2Fo%3Aoracle%3Alinux%3A8/https://github.com/ComplianceAsCode/content
```

Not all distrubutions are currently supported. If your distribution is missing, all you have to do is
to create a new Dockerfile in this [folder](https://github.com/ComplianceAsCode/content/tree/master/Dockerfiles) respecting the naming convention `test_suite-<product_name>`.
This container image must be able to run test scenarios from Automatus, so you might get inspired by other similar Dockerfiles from that folder.

## Content Navigator

The environment comes with the Content Navigator extension pre-installed, which means you can benefit from all the
features that the extension provides. For more details you can check this [blogpost](https://complianceascode.github.io/template/2019/12/19/content-navigator-a-vscode-extension.html) or the [extension's project page](https://github.com/ggbecker/content-navigator/)

## Running Tests

In order to run the test scenarios, simply press `Ctrl+F5` while having a `rule.yml` (or any other remediation file or test scenario) opened in the text editor that it will start running tests in [`Rule`](https://complianceascode.readthedocs.io/en/latest/tests/README.html#rule-based-testing) mode. The Content Navigator extension provides a feature that detects the rule ID and will ensure it is propagated to the command line that executes the test.

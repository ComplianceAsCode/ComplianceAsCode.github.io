---
layout: post
title: "Gitpod.io Integration"
categories: template
author: Gabriel Becker
---

## Introduction

Now you can take advantage of `gitpod.io` to create fresh development environments of ComplianceAsCode/content.
You just need a github account to start benefiting from this integration.


### Where It Can Be Used

You can use the tool to contribute to the project or you can use the environment to review pull requests.

The following link will create a ready to develop environment with all the required dependencies installed and a fully
working test environment using containers:

https://gitpod.io/#https://github.com/ComplianceAsCode/content

While reviewing pull requests in the project, a comment with a button will be displayed and the link
will create an environment that contains the development environment plus the code of the particular
pull request. For example, see this [comment](https://github.com/ComplianceAsCode/content/pull/9040#issuecomment-1167294750).

## Testing Environment

A fresh testing environment should be ready to be used after the environment is deployed.

Run configurations are also available, which means you can simply activate the Run configurations as you would normally do in your
Visual Studio code instance.

At this moment, only Fedora container images are available. The intent is to expand the scope to other distributions.
If you are insterested in contributing new images, please do so by submitting your Dockerfile here: https://github.com/ComplianceAsCode/content/tree/master/Dockerfiles

These images can be used in the `gitpod.io` integration to generate environment specifically for you distribution.

The following parameters are available to customize the deployment:


`PRODUCT`: The product to be built (./build_product $PRODUCT)

`CONTAINER`: The Dockerfile to be used when creating the testing container (the following pattern: Dockerfiles/test_suite-$CONTAINER). This defaults to `PRODUCT` if not set.

`CONTAINER_VERSION`: Optional. If the Dockerfile supports defining the distribution version same as defined in [test_suite-fedora](https://github.com/ComplianceAsCode/content/blob/bf5f0381df44a2079ec356d683c091b1b828931f/Dockerfiles/test_suite-fedora#L2). 

`CPE`: CPE identifier of the testing environment. You can run `cat /etc/os-release | grep CPE_NAME` to obtain this information.

Below is an example of how to invoke `gitpod.io` with parameters:

```
https://gitpod.io/#PRODUCT=rhel8,CONTAINER=fedora,CONTAINER_VERSION=fedora:36,CPE=cpe:/o:fedoraproject:fedora:36/https://github.com/ComplianceAsCode/content
```

## Content Navigator

The environment comes with the Content Navigator extension pre-installed, which means you can benefit from all the
features that the extension provides. For more details you can check this [blogpost](https://complianceascode.github.io/template/2019/12/19/content-navigator-a-vscode-extension.html) or the [extension's project page](https://github.com/ggbecker/content-navigator/)

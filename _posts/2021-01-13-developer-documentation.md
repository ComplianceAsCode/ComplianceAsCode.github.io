---
layout: post
title: "Hosting the Developer Documentation on ReadTheDocs.org"
categories: template
author: Gabriel Becker
---

### Introduction

Recently we have made our project's documentation available on [Read the Docs](https://ReadTheDocs.org) platform. The documentation is available at [complianceascode.readthedocs.io](https://complianceascode.readthedocs.io).

Hosting the documentation on `Read the Docs` brings many benefits to the project. The documentation is easier to navigate, easier to search and it's better structured to support different documentation types in a single place.

### History

Previously the common way of accessing the documentation was to browse through files hosted on Github's project page or on a local copy of the project's repository. These documentation files are usually written in AsciiDoc or Markdown. Navigating and/or searching was essentially difficult and making the documentation available on an online platform seemed natural. The choice of `Read the Docs` was quite simple to make since it's a free and well known platform.

### Read the Docs

`Read the Docs` is a documentation hosting platform which is Open Source and provides many ways to host your project's documentation. It mainly supports `Sphinx` format, but using additional plugins it's possible to render `Markdown` format. Unfortunately `AsciiDoc` is not officially supported and some parts of the developer documentation had to be ported to Markdown first.

### Project's Documentation

[ComplianceAsCode content project](https://github.com/ComplianceAsCode/content/) contains many documentation pieces spread all over the project, there is the [developer documentation}(https://github.com/ComplianceAsCode/content/tree/master/docs/manual/developer) which contains instructions on how to build the project and many sections related to development of the project. There is the [testing documentation](https://github.com/ComplianceAsCode/content/blob/master/tests/README.md) which contains information about the SSG Test Suite and about many other test aspects of the project. The build system is written in `Python` for the most part and source code documentation can be found on [Python source files](https://github.com/ComplianceAsCode/content/tree/master/ssg). `Jinja` macros also contain DocString alike documentation which can be found on [Jinja source files](https://github.com/ComplianceAsCode/content/tree/master/shared).

### Read the Docs Integration

At the moment of writing this article, the [complianceascode.readthedocs.io](https://complianceascode.readthedocs.io) contains five main categories:

- Developer Guide
- Jinja Macros Reference
- Python Modules Reference
- SSG Test Suite Developer Guide
- Release Tools Documentation

Each of them reflects the contents of their respective documentation present on [ComplianceAsCode content project](https://github.com/ComplianceAsCode/content/). Whatever change is made in the project's documentation is reflected in the [complianceascode.readthedocs.io](https://complianceascode.readthedocs.io) automatically. And for each new release of [ComplianceAsCode content project](https://github.com/ComplianceAsCode/content/), a new tagged version of the documentation is created.

### Conclusion

Whether you are getting acquainted with the project or are a long term developer, this documentation transition to `Read the Docs` has brought many benefits for contributors. The documentation became much more accessible and well organized. Contributors to the project can now efficiently consult project's documentation and focus more on developing security content.

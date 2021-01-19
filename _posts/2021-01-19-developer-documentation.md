---
layout: post
title: "Hosting the Developer Documentation on ReadTheDocs.org"
categories: template
author: Gabriel Becker
---

### Introduction

Recently we have made our project's documentation available on [Read the Docs](https://ReadTheDocs.org) platform. The documentation is available at [complianceascode.readthedocs.io](https://complianceascode.readthedocs.io).

`Read the Docs` is a documentation hosting platform that is Open Source and provides many ways to host your project's documentation. It mainly supports `Sphinx` format, but using additional plugins it is possible to render `Markdown` formatted content. Unfortunately `AsciiDoc` is not officially supported and some parts of the developer documentation had to be ported to Markdown first.

Hosting the documentation on `Read the Docs` brings many benefits to the project. The documentation is now:

- Easier to navigate;
- Easier to search;
- Better structured to support different documentation types in a single place.

### History

Previously the common way of accessing the documentation was either to browse through files hosted on Github's project page or check a local copy of the project's documentation. These documentation files are usually written in `AsciiDoc` or `Markdown`. Navigating and/or searching was essentially difficult and making the documentation available on an online platform seemed natural. The choice of `Read the Docs` was quite simple to make since it's a free and well-known platform.
### Read the Docs Integration

ComplianceAsCode content project contains many documentation pieces spread all over the project and at the moment of writing this article, the [complianceascode.readthedocs.io](https://complianceascode.readthedocs.io) contains five main documentation categories:

- [Developer Guide](https://github.com/ComplianceAsCode/content/tree/master/docs/manual/developer)
- [Jinja Macros Reference](https://github.com/ComplianceAsCode/content/tree/master/shared)
- [Python Modules Reference](https://github.com/ComplianceAsCode/content/tree/master/ssg)
- [SSG Test Suite Developer Guide](https://github.com/ComplianceAsCode/content/blob/master/tests/README.md)
- [Release Tools Documentation](https://github.com/ComplianceAsCode/content/blob/master/release_tools/README.md)

Each of them reflects the contents of their respective documentation present on ComplianceAsCode content project. Whenever a change is made in the project's documentation, it is automatically reflected in the [complianceascode.readthedocs.io](https://complianceascode.readthedocs.io). And for each new release of ComplianceAsCode content project, a new tagged version of the documentation is created.

#### Build the Documentation Locally

If you would like to build the documentation locally, whether to access it offline or to contribute back, you can do it by following these instructions:

- [Install Sphinx Packages](https://complianceascode.readthedocs.io/en/latest/manual/developer/02_building_complianceascode.html#installing-build-dependencies)
- [Generate HTML documentation](https://github.com/ComplianceAsCode/content/tree/master/shared)

### Conclusion

Whether you are getting acquainted with the project or are a long term developer, this documentation transition to `Read the Docs` has brought many benefits. Contributors can now effectively consult the documentation and focus more on developing security content. Go ahead and check it out at [complianceascode.readthedocs.io](https://complianceascode.readthedocs.io) if you haven't done so and don't forget to bookmark it.

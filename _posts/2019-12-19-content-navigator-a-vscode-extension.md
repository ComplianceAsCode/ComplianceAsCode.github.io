---
layout: post
title: "Content Navigator - A Visual Studio Code Extension to Help Users to create Security Content"
categories: template
author: Gabriel Becker
---

In [ComplianceAsCode content project](https://github.com/ComplianceAsCode/content/), there are hundreds of rules (to be exact 1617 of them as per 2019-12-19) which are shared among many products and each rule can contain different content, for example, remediation (eg. Bash, Ansible, Anaconda, etc.) and [OVAL](https://oval.mitre.org/) checks. If you are not familiarized with the structure of the project it might be hard to navigate between those files. [Content Navigator](https://github.com/ggbecker/content-navigator/) is a Visual Studio Code Extension which helps the user to navigate between content providing shortcuts and additionally provides content awareness through auto completion of existing rules while editing profiles and code snippets which contain stubs for content such as fields required when creating a new rule and also available templates.

### Visual Studio Code

[Visual Studio Code](https://code.visualstudio.com/) often referenced as VSCode: "is a lightweight but powerful source code editor which runs on your desktop and is available for Windows, macOS and Linux." It can be easily installed by following the instructions from their main website: [Visual Studio Code](https://code.visualstudio.com/)

Additionally to the rich set of built-in features that VSCode provides, it is possible to install `extensions` which extend its capabilities. There are many useful extensions available already. You can skim through them on their [marketplace](https://marketplace.visualstudio.com/VSCode).

To install Content Navigator you can either follow the instructions from [content-navigator marketplace web page](https://marketplace.visualstudio.com/items?itemName=ggbecker.content-navigator) or search for `content-navigator` under the `Extensions` tab on VSCode (Press `Ctrl+Shift+X` to open the `Extensions` tab).

### Content Navigator

When the setup is done, open the ComplianceAsCode content folder on VSCode and you can start to use Content Navigator extension. If you right click on any opened file you should see something like:

![image](/assets/images/right_click_menu_red_stroke.png)

These features usually work when you have text selected or there is content stored in the clipboard. The extension will try to identify the content corresponding to the provided text and will open the correct file if a match is found.

So let's say you open the file `rhel8/profiles/ospp.profile` and select the text `enable_fips_mode`, by activating the option `Open Rule` from the right click menu, the corresponding `rule.yml` file should be opened similar to the following:

![image](/assets/images/open_rule_enable_fips_mode.gif)

The same is valid to other kinds of content, OVAL checks, Bash remediation, etc. If the content exists it will open the corresponding file based on the requested option.

#### Open Content

As described previously, you can open content by selecting a rule id or storing a rule id in the clipboard (Note: the clipboard always take precedence) and then activating an option through the right click menu or by pressing a combination of keys. Current content types supported are:

- Rule: `Ctrl+Alt+R`
- OVAL: `Ctrl+Alt+O`
- Bash: `Ctrl+Alt+B`
- Ansible: `Ctrl+Alt+A`
- Anaconda: `Ctrl+Alt+N`
- Puppet: `Ctrl+Alt+P`

Note that the extension also supports navigating between content corresponding to the current opened file. The extension tries to identify the rule id corresponding to the current opened file and opens the file corresponding to the rule id found. But always keep in mind that the clipboard takes precedence and if you have a rule id stored there it will open the content based on that.

#### Code Snippets

Code snippets is a great feature provided by VSCode and allows you to reduce boilerplate work and increase productivity. Content Navigator provides code snippets for Compliance As Code content project. Currently, the main focus is the templating system where all available templates are supported. The new templating system mechanism, which is described in this [blog post]({% post_url 2019-10-16-new-templating-system %}), allows you to use a template by simply adding a few lines to the `rule.yml` file. But the question is, which template should I use and which parameters I can use to fine tune the template? Code snippets can help you in this situation, the following image illustrates the situation:


![image](/assets/images/code_snippets_template.gif)

Note that a skeleton of the template is made and each part contains the official documentation describing its purpose and which values it takes. To activate the box with available templates simply press `Ctrl+Space` when editing a `rule.yml` file.

There are many code snippets for template, they all come from [ComplianceAsCode/content Templates Section of Developer Guide](https://github.com/ComplianceAsCode/content/blob/master/docs/manual/developer_guide.adoc#732-list-of-available-templates) and are also briefly listed in the [Content Navigator Documentation](https://github.com/ggbecker/content-navigator/blob/master/README.md). Go ahead and explore them!

#### Other Features

Check the [Documentation](https://github.com/ggbecker/content-navigator/blob/master/README.md) out to find out more about other features available in Content Navigator. I am sure that they will be helpful for you at some point.

### Conclusion

By the time I started to contribute to Compliance As Code content project, I felt that it was difficult to find something within the project because it is a big project and it involves many aspects. I found myself repeating various commands over and over, mainly when navigating between files. Then, I started thinking on how to improving the situation, at first I thought it would not save me much time but nowadays it has proved to be a tool with significant benefit to my daily work. There are still plenty of room for improvement ([See issues on github](https://github.com/ggbecker/content-navigator/issues)) and I plan to keep improving Content Navigator to make our life easier when contributing to Compliance As Code content project.

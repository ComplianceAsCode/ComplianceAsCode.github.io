---
layout: post
title: "Defining rule applicability using platform expressions"
categories: template
author: Jan Černý
author_url: https://github.com/jan-cerny
---

Not every XCCDF rule makes sense on every system.
For example, think of rules that configure Kerberos settings: if the system doesn't have Kerberos installed, these rules don't make sense and shouldn't be evaluated.
Similarly, rules that configure the zIPL bootloader make sense only if the evaluated system runs on the s390x architecture.
If the rules like that would be evaluated it would cause confusing compliance reports.

Our project generates a single SCAP source data stream for a product (eg. RHEL 9).
The data stream is used on all variants of the product and all possible setups.
We don't ship different SCAP source data stream files based on architecture, or based on the package selection.
Therefore, we need solve the problem of not always usable rules by limiting their applicability.
All rules are always available in the generated SCAP source data stream, but some of them can be evaluated by OpenSCAP as "notapplicable" if some applicability condition isn't met.

This is achieved by adding so-called platforms to the rules.
The platforms provide applicability checks for the rules.
Technically, we implement the applicability checks using the Common Platform Enumeration (CPE).
This technology allows SCAP scanners to identify properties of the given system, such as its operating system version or presence of installed software packages.

Recently, we have introduced many improvements in ComplianceAsCode that improve support for defining rule applicability and allow content authors to specify the applicability more easily and more flexibly.

In this blog post, we will first briefly introduce the concept of platforms for checking the rule applicability.
Then, we will show the new improvements of this concept and demonstrate their usage.

## Basic platforms usage

The platform definitions are located in the `/shared/applicability` directory.
Each platform definition is a YAML file which defines a platform.
The ID of the platform is the YAML file's base name.

For example, the `uefi` platform is defined in `/shared/applicability/uefi.yml` and it looks like this:

```
name: cpe:/a:uefi
title: System boot mode is UEFI
check_id: system_boot_mode_is_uefi
bash_conditional: '[ -f /sys/firmware/efi ]'
ansible_conditional: '"/boot/efi" in ansible_mounts | map(attribute="mount") | list'
```

To use the platform in a rule, add a `platform:` key to the `rule.yml` in the rule directory, for example:

```
platform: uefi
```

The outcome is that when the generated SCAP source data stream is evaluated by OpenSCAP scanner and the system doesn't have UEFI, the result of the rule evaluation will be "notapplicable".
Therefore, the user won't receive any false positive about UEFI misconfiguration in his report.
Moreover, if the Bash or Ansible remediations of this rule are executed on such system, they won't do any action because they will be wrapped in an applicability check as well.

In our example, adding the `platform: uefi` line to the `rule.yml` file will cause that the generated XCCDF rule in the built SCAP source data stream will contain an `<xccdf-1.2:platform>` element which references a CPE item `cpe:/a:uefi` which references a OVAL check `system_boot_mode_is_uefi`.
This check is defined in the `/shared/applicability/oval` directory.
Moreover, the Bash remediation will be wrapped by the condition specified in the `bash_conditional` key.
Similarly, the Ansible task will be extended by a when statement containing the condition specified in the `ansible_conditional` key.

There exist multiple useful platforms in the ComplianceAsCode repository that are based on architecture, presence of files or packages, containers, partitions and they are used in many rules across multiple products.

The basic platform definitions are available for content authors for quite some time, so in the rest of the blog post, we will focus on the new features and how they can help you define rule applicability scope.

## Templated platforms

The first improvement that we made is that platform definitions can be now templated.
This is convenient for adding platforms that differ only in their parameter.
We already have 3 templated platforms ready to use in the project: `package`, `linux_os` and `partition`.

For example, to limit the rule applicability to systems where sudo is installed, add the following platform definition to the `rule.yml` of your rule:

```
platform: package[sudo]
```

As you can see, the argument for the platform is put into square brackets after the name of the platform.
The list of platform arguments is explicitly set in the platform definition file (in the `/shared/applicability` directory).
If you need to add a new argument, simply add the argument and its value to the platform file.

Here is how the `package` platform definition (located in `shared/applicability/package.yml`) looks like:

```
name: "cpe:/a:{arg}:{ver_specs_cpe}"
title: "Package {pkgname} is installed"
versioned: true
template:
  name: platform_package
args:
  audit:
    pkgname: audit
  chrony:
    pkgname: chrony
  gdm:
    pkgname: gdm

( ... snip ... )
```

Using templated platforms avoids code duplication and prevents inconsistencies.

It's also easy to create a new platform template.
The templated platforms use the same template mechanism that is used for checks and remediations, so the content authors are already familiar with using it.
Take a look for inspiration for example to the `/shared/templates/platform_package` directory.

## Versioned platforms

Sometimes, you might need to write a rule for some configuration compliance setting which is configuring a new feature that is available only on the new version of the software.
That means that you want your rule to be evaluated only if that specific version or a newer version is installed.
With older versions of the installed software, you want your rule to end up as not applicable.

To enable creating these rules our `package` and `os_linux` platforms now support versions.
For example, you can use in your rules:

```
platform: package[usbguard]>2.0
```

```
platform: os_linux[rhel]>=8.6
```

If the first platform definition is used in a rule, the rule will be applicable if the package usbguard is installed and is newer than 2.0.
It will check RPM packages on RPM systems and DEB packages on Debian systems.
The second platform will be true on RHEL 8.6 and newer, and the evaluation will be based on reading `/etc/os-release`.

At the first sight, it looks like that the `package` platform is the go-to platform that will be used in most cases.
But in general, we recommend to derive rule applicability from operating system release version instead of basing that on a version of a specific package.
The reason is that in Linux distributions such as RHEL the features and fixes often get backported to older versions, so the actual feature set might not correspond to the upstream version of the project, but rather depend on the actual contents of the currently shipped package in the given Linux distribution.
Therefore, the package versioned platform might not work or can differ in different distributions.
It's better to track down the contents of the packages and identify the version of the system where this package has been shipped, and use the `os_linux` platform instead.

Also, it's tricky to create a correct version expression due to existence of epoch and release components of the version in the RPM or DEB package versions.
For example, upstream package `foo` version `1.2.3` can be packaged in RHEL 8 as `foo-1.2.3-1.el8`, but differently on Debian.

## Using logical expressions in platform definitions

The recently introduced support for CPE applicability language allows content authors to combine multiple platforms into complex platform expression using logical operators.
You can use the common operators (and, or, not, ...) and parenthesis.

Imagine that your rule should be applicable only on non-containerized systems that have either chrony or ntp package installed.
You can now simply write the following into the `rule.yml` of that rule and the build system will generate the applicability check code for you and will insert it to the rule.

```
platform: machine and (package[chrony] or package[ntp])
```

This isn't an artificial example, you can see this in practice in the rule [chronyd_or_ntpd_specify_remote_server](https://github.com/ComplianceAsCode/content/blob/42d9d7f48048d60ee17ebc6c1d2cf3420ac4326b/linux_os/guide/services/ntp/chronyd_or_ntpd_specify_remote_server/rule.yml#L89).

The outcome is that the generated rule will contain a platform check composed from the individual checks and also the remediations will be extended by a combined condition.

Of course, you can use versioned platforms in the logical expressions as well.

## Next steps

We would like to leverage these new features to be able to catch differences between different minor versions of the same product.
We envision being able to do this directly in the upstream repository.
That means we will no longer need to distribute different versions of SCAP source data streams that contain different patches for different minor versions of RHEL and/or other Linux distributions.
Instead, there can be a single data stream that is usable across all minor versions.
Stay tuned.

## Conclusion

The rule applicability can be defined using flexible platform expressions.
This allows content authors to limit rule applicability based on presence of package, version of package, version of the operating system, architecture and other artifacts and also based on combination of these factors.
For more details on the applicability, read the [Applicability](https://complianceascode.readthedocs.io/en/latest/manual/developer/06_contributing_with_content.html#applicability-of-content) section in project documentation.

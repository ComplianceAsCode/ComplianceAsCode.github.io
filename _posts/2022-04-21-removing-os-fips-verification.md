---
layout: post
title: "Removing the OS FIPS Certified Verification from the Enable FIPS Mode rule"
categories: template
author: Gabriel Becker
img: thumbnail/enable_fips_mode.png
---


The [ComplianceAsCode content project](https://github.com/ComplianceAsCode/content/) contains the rule [enable_fips_mode](https://github.com/ComplianceAsCode/content/blob/master/linux_os/guide/system/software/integrity/fips/enable_fips_mode/rule.yml) that checks if a system has the FIPS mode enabled in the Operating System (OS).
Enabling the FIPS mode means that the system's cryptographic modules are running in a mode that only NIST approved algorithms, ciphers and everything related to cryptography are allowed to be used.
But, it doesn't mean that the system's cryptography modules have met all the FIPS 140-2 requirements and it doesn't mean that the system has received FIPS 140-2 certification by NIST as well.

So for example, when the rule is reported as pass after a scanning, it means that the cryptographic modules are running in FIPS mode.
It doesn't mean that the OS is FIPS certified and additional validation must be done to attest the system's certification.
This validation is entirely out of the scope of this project.

The rule `enable_fips_mode` was known for having an extended check that comes from the rule [installed_OS_is_FIPS_certified](https://github.com/ComplianceAsCode/content/blob/master/linux_os/guide/system/software/integrity/certified-vendor/installed_OS_is_FIPS_certified/rule.yml).
This rule contains the list of FIPS 140-2 certified Operating Systems meaning that the rule `enable_fips_mode`
had only passed if the OS was included in this [list](https://github.com/ComplianceAsCode/content/blob/master/linux_os/guide/system/software/integrity/certified-vendor/installed_OS_is_FIPS_certified/oval/shared.xml).

Nevertheless, you can still use the rule `installed_OS_is_FIPS_certified` combined with `enable_fips_mode` in the same profile to have similar results as previously.

<!-- The community has already tried to go in this direction in the past but it was
rejected. For more details you can check these links:
[PR#4920](https://github.com/ComplianceAsCode/content/pull/4920) and [Issue#4917](https://github.com/ComplianceAsCode/content/issues/4917). -->

We have then decided to ease the situation and remove this extended check from
the `enable_fips_mode` that will only check for the technical aspect of FIPS
enablement in the Operating System. Additional verification is still needed to
make sure that the OS is certified by NIST and that can be done via using the
rule `installed_OS_is_FIPS_certified` or checking on official [NIST certified
systems database](https://csrc.nist.gov/projects/cryptographic-module-validation-program/validated-modules/search).

This change was done in
[PR#8255](https://github.com/ComplianceAsCode/content/pull/8255) and is
available since release 0.1.61.



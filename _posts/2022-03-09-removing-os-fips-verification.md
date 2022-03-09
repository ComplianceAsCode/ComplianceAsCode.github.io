---
layout: post
title: "Removing the OS FIPS Certified Verification from Enable FIPS Mode rule"
categories: template
author: Gabriel Becker
---


The [ComplianceAsCode content
project](https://github.com/ComplianceAsCode/content/) contains the rule [enable_fips_mode](https://github.com/ComplianceAsCode/content/blob/master/linux_os/guide/system/software/integrity/fips/enable_fips_mode/rule.yml) that checks if a
system has the FIPS mode enabled in the Operating System (OS). This means that cryptographic modules are
running in a mode that only NIST approved algorithms, ciphers and everything related
to cryptography are allowed to be used. BUT, it doesn't necessarily mean that
it is a NIST certified OS (cryptographic modules) that meets the
FIPS 140-2 requirements. For that to happen, the OS has to undergo
through NIST's FIPS 140-2 certification process.

This rule was known for having an extended check that comes from the rule
[installed_OS_is_FIPS_certified](https://github.com/ComplianceAsCode/content/blob/master/linux_os/guide/system/software/integrity/certified-vendor/installed_OS_is_FIPS_certified/rule.yml)
that contains the list of FIPS certified Operating Systems (once again,
certified cryptographic modules). So this means the rule `enable_fips_mode`
would only pass if the OS is included in this
[list](https://github.com/ComplianceAsCode/content/blob/master/linux_os/guide/system/software/integrity/certified-vendor/installed_OS_is_FIPS_certified/oval/shared.xml).

The community has already tried to go in this direction in the past but it was
rejected. For more details you can check these links:
[PR#4920](https://github.com/ComplianceAsCode/content/pull/4920) and [Issue#4917](https://github.com/ComplianceAsCode/content/issues/4917).

We have then decided to ease the situation and remove this extended check from
the `enable_fips_mode` that will only check for the technical aspect of FIPS
enablement in the Operating System. Additional verification is still needed to
make sure that the OS is certified by NIST and that can be done via using the
rule `installed_OS_is_FIPS_certified` or checking on official [NIST certified
systems database](https://csrc.nist.gov/projects/cryptographic-module-validation-program/validated-modules/search).

This change was done in
[PR#8255](https://github.com/ComplianceAsCode/content/pull/8255) and will be
available in the next release 0.1.61.



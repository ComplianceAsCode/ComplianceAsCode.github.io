---
layout: post
title: "Using Thin Data Streams in ComplianceAsCode"
categories: template
author: Jan Rodák
author_url: https://github.com/Honny1
---

## Introduction

Last quarter, the ComplianceAsCode project introduced thin data streams which is a great enhancement for every developer working on security content in the project.
A thin data stream is a SCAP source data stream that contains just a single rule.
It contains only data related to the given rule and doesn't contain content related to any other rule.
Thin data streams have an average size of approximately 62 kB.

## Where It Can Be Used

Thin data streams contain a minimal amount of content for the rule or profile testing.
It will allow to significantly shorten the author-test-refactor-test-... loop.
The thin data streams are easy to read due to their small size.

Thin data streams with minimal content will be faster to create and also faster to test using Automatus.
Today, testing uses a normal data stream, which the scanner must read.
This takes a lot of time when one has selected one rule to be tested and the scanner analyzes the whole data stream.
Thin data streams are much smaller than normal data streams and take less time to load.
Generating thin data streams for all rules for a given product takes about the same time as generating a normal data stream.

## How Fast Is It Actually

To compare the performance between thin and normal data streams we used profiling capability of the build system.
To test differences in the built Fedora product and for the thin data streams, the rule `enable_fips_mode` was selected.

Building a normal data stream takes 1 minute 53.4 seconds and building a thin data stream with one rule `enable_fips_mode` takes 1 minute 39.4 seconds with is very similar time.
But more exciting information is that using a build with `--thin` flag to build a thin data stream for each rule takes time: 2 minute 42.6 seconds.
This time is about one minute more but it built 1874 thin data streams.

Now, let's demonstrate the impact of data stream size on its evaluation in OpenSCAP.
This command was used to benchmark the OpenSCAP scanner:

```bash
perf stat -e cpu-clock -r 10 oscap xccdf eval --rule "xccdf_org.ssgproject.content_rule_enable_fips_mode" --results-arf arf.xml ./build/ssg-fedora-ds.xml
```

The only difference is the different size of the data stream.

This is the result of a normal data stream:

```bash
Performance counter stats for 'oscap xccdf eval --rule xccdf_org.ssgproject.content_rule_enable_fips_mode --results-arf arf.xml ./build/ssg-fedora-ds.xml' (10 runs):

          4 634,43 msec cpu-clock:u           #    0,992 CPUs utilized   ( +-  4,11% )

             4,670 +- 0,197 seconds time elapsed  ( +-  4,22% )
```

This is the result of a thin data stream:

```bash
Performance counter stats for 'oscap xccdf eval --rule xccdf_org.ssgproject.content_rule_enable_fips_mode --results-arf arf.xml ./build/ssg-fedora-ds.xml' (10 runs):

          2 257,85 msec cpu-clock:u           #    0,998 CPUs utilized      ( +-  0,32% )

           2,26231 +- 0,00759 seconds time elapsed  ( +-  0,34% )
```

Here is a demonstration of the improvement of running the Automatus test suite.
This command was used to benchmark the Automatus script:

```bash
perf stat -e cpu-clock -r 10 ./automatus.py rule --libvirt qemu:///session test-suite-fedora audit_rules_privileged_commands
```

The only difference is the different size of the data stream.

This is the result when using a normal data stream:

```bash
Performance counter stats for './automatus.py rule --libvirt qemu:///session test-suite-fedora audit_rules_privileged_commands' (10 runs):

         45 141,65 msec cpu-clock:u          #    0,104 CPUs utilized     ( +-  1,51% )

            432,24 +- 4,39 seconds time elapsed  ( +-  1,02% )
```

This is the result when using a thin data stream:

```bash
Performance counter stats for './automatus.py rule --libvirt qemu:///session test-suite-fedora audit_rules_privileged_commands' (10 runs):

         15 917,99 msec cpu-clock:u          #    0,066 CPUs utilized     ( +-  1,43% )

            239,53 +- 2,44 seconds time elapsed  ( +-  1,02% )
```

As you can see above, scanning a single rule with a thin data stream takes half the time.
Using a thin data stream with the Automatus script shows a significant time reduction of about 3 minutes and 20 seconds.
When running test scenarios for all rules.
This, along with batch generation of thin data streams, can significantly reduce the execution time of a test suite.

## How To Build Thin Data Streams

Use the `build_product` script with the `--rule-id` or `--thin` flags to start the thin data stream generation.
The `--thin` flag generates a thin data stream for each rule in the product.
The `--rule-id` flag generates a thin data stream for a single rule whose ID was specified as the value after the flag.

For example, this command will generate a thin data stream with one rule:

```bash
./build_product fedora --rule-id enable_fips_mode
```

For example, this command generates thin data streams for all rules for the Fedora product and the thin data streams are stored in the `./build/thin_ds` directory:

```bash
./build_product fedora --thin
```

## How Thin Data Streams Are Generated

To be able to generate thin data streams the build system was reworked to iterate through the profiles and include only rules that are selected in profiles.
To generate a thin data stream, a simple profile that contains only one rule is created internally.
The simple profile replaces the product’s profiles. The generation continues as in a normal build.

The normal build has been enhanced to remove unnecessary parts of the data stream.
This allows the normal data stream to benefit from the improvements.
Unfortunately, this does not have a big effect on the size of the data stream, because it contains a lot of rules, but thin data streams that contain only one rule have an average size of about 62 kB.

An important internal enhancement that enables the creation of thin data streams is the implementation of the OVAL object model.
This allows the build system to filter OVAL Definitions and other OVAL components and create minimal OVAL documents for a given OVAL Definition.

Thin data streams do not contain any OCIL.
However, in normal data streams, OCIL was reduced only for rules selected in the profiles.

## Conclusion

Thin data streams are easy to read due to their small size and faster to process using the OpenSCAP scanner and test suite, this will allow to significantly shorten the author-test-refactor-test-... loop, and will change the way content is developed.
They are also a prerequisite to further improvements and many other applications.
The execution time of an openSCAP scanner with a thin data stream is about half that of a normal data stream and builds take a similar amount of time as a normal data stream.
They also speed up the Automatus script runtime by approximately 3 minutes. This is a big step towards a faster test suite.

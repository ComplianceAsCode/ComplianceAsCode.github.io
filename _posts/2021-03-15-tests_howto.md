Testing of rules and remediations
=================================

How do we make sure that rules in the ComplianceAsCode repository are correct?
Before we answer that, let's take a look at the big picture.
The part of a rule that is a code of some kind can include OVAL definition, a Bash snippet, and a set of Ansible tasks.

OVAL is not a very expressive language, and it is not very readable either.
Bash is a shell in which you can write simple programs.
Robustness of those programs and readability of the code are not a strong aspects either.
Finally, Ansible is primarily a configuration management software, and hardening is a more delicate use case, so the readability of a rule's Ansible content can suffer from it.
So being sure that a given rule is written correctly is not trivial.

However, we are not finished here.
The ComplianceAsCode repository hosts more than one thousand rules.
Many of them are templated, i.e. they are slight variations of one principle, but there are still hundreds that are crafted by hand.
Therefore, making sure that even rules that are used by a given product are correct is tough to scale - that means verifying tens to hundreds of rules.

So how do we do it in the project?
As you can learn very quickly by visiting the [tests](https://github.com/ComplianceAsCode/content/tree/master/tests#readme) folder, the project features a test suite.
There is everything in it, but let's just skim through the simplest use cases that will likely get you covered:


The Test Suite
--------------

What does the test suite do?
It is able to verify that a check passes or fails as expected.
Moreover, in case that if it fails, it tests that a remediation is able to fix the system, so the check on the fixed system passes.

It does so by using virtualization (containers or VMs) to set up the virtualized system.
You can see the `tests` directory in rule with scripts - each script represents one test scenario, which sets up the system to a certain state that should be picked up by the scanner.
A test scenario may be tied to a particular product or profile, but if the rule is about correct contents of a configuration file, such scenario can be applied to any product.

The test suite executes the scenario script, and then it performs the scan.
If the scanner reports a failed rule, and the scenario was supposed to fail, remediation is applied, and another scan is conducted.
Results are then reported to the command-line, and saved to the log file.

Therefore, the test suite performs these tasks:

1. Environment preparation.
2. Initial scan - this can be the last thing if the test scenario is supposed to pass.
3. Remediation - in case of `oscap` remediation, this step includes the final scan as well.
4. Scan of remediated system.


Testing as an unprivileged user
-------------------------------

Since `podman` became a standard part of Linux distributions, one can build and run containers locally without additional privileges, so let's see how such test can look like.
First of all, we will need a test environment.
The project already contains predefined recipes, so let's leverage them, and build the test container:

```shell
$ tests/build_test_container.sh
```

That's it, the script builds a Fedora-based test container image called `ssg_test_suite` by default, which can get us quite far, as you will see.
The script offers options to customize its behavior - simply call `tests/build_test_container.sh -h` to see how.
If the selection of flavors caught your eye, those have to be in line with contents of the [Dockerfiles](https://github.com/ComplianceAsCode/content/tree/master/Dockerfiles) directory, specifically to `test_suite-...` recipes.
However, you can use a Fedora-based image to test many rules even if you are a Debian content developer.

Next, we build the content that we would like to test - so let's dare and test a rule in the RHEL7 context.
We do it to demonstrate the low sensitivity of the tested content to its environment - RHEL7 is surely not Fedora.
You can go ahead and use a different product further from Fedora, just make sure that its benchmark includes the `accounts_tmout` rule that we are going to use as an example.

```shell
$ ./build_product rhel7
```

... and we have the datastream!
Now let's test a rule that is text-based, cross-platform and easy to follow - the famous [accounts_tmout](https://github.com/ComplianceAsCode/content/blob/master/linux_os/guide/system/accounts/accounts-session/accounts_tmout/rule.yml) one:

```shell
$ tests/test_rule_in_container.sh --dontclean accounts_tmout
...
INFO - Script line_not_there.fail.sh using profile (all) OK
INFO - Script multiline.fail.sh using profile (all) OK
INFO - Script wrong_value.fail.sh using profile (all) OK
INFO - Script correct_value.pass.sh using profile (all) OK
```

...

So what happened?
The test suite executed `oscap` in the Fedora container, but two things had to be adjusted to make it work:

1. The applicability of the datastream is extended to include the system used for the actual test - in this case Fedora.
The content still corresponds to the system that the datastream has been compiled for, in this case that would be the RHEL7 content, but it could be SLE or Ubuntu content as well.
2. Our example rule is marked as machine-only, i.e. not applicable in container environment, which would the test of the rule in a container impossible.
The test suite removes this applicability restriction from the rule and its remediations.
Some rules don't make sense in a container in production, but technically, they can be formally tested and remediated the same way as they would be on a bare-metal system.

That sounds like the wrapper did a bit of a black magic, but you can always pass the `--dry-run` argument to reveal what it did:

```shell
$ tests/test_rule_in_container.sh --dry-run --dontclean accounts_tmout
python3 /.../tests/test_suite.py rule --remove-machine-only --dontclean --add-platform fedora --container ssg_test_suite -- accounts_tmout
```

Therefore, you can execute the actual test suite by yourself, and fine-tune the execution as soon as you get comfortable with its usage.
Some optional arguments are not needed - `--scenario .*` and `--remediate-using oscap` just set their default values.

The test suite reports results in the console, but you can see greater details in the log directory.
Logs are stored in the `logs` folder that is located in the current directory - each test run has its own log folder.
Navigate to it, and open the most recent log folder that contains details of the test run.

There are some notable files in there:

- `results.json`: A machine-readable summary of the test run.
You can check it out - it records results in individual test phases and other metadata.
It's worth mentioning that the `tests/analyze_results.py` script can help you to deal with a huge number of scenarios tested, or with a series of tests that were conducted over a period of time.
- `<long filename>.html` - OpenSCAP HTML report of a test phase of a scenario.
Normally, it is deleted, so you have to pass `--dontclean` to the test suite to get it.
It contains OVAL details, so you can debug the rule if the scenario ends unexpectedly.
- `<long filename>.verbose.log` - OpenSCAP log of a test phase of a scenario.
In case that you don't understand how come that the scanner collected or missed something, this is the place to look.
- `.log` files with short names: Those are high-level logs created by the test suite.
If the test suite has problems connecting to the tested system, or if the environment fails to prepare, that's where one can find out more.


Test scenarios
--------------

Let's take a closer look at those scenarios.
You can find them in the [tests subfolder of the respective rule](https://github.com/ComplianceAsCode/content/tree/master/linux_os/guide/system/accounts/accounts-session/accounts_tmout/tests)
Consider, for instance, [comment.fail.sh](https://github.com/ComplianceAsCode/content/blob/master/linux_os/guide/system/accounts/accounts-session/accounts_tmout/tests/comment.fail.sh)

There are following things that we can tell about that scenario:

- Its filename ends by `fail.sh` - that means that the scenario is supposed to make the rule scan to fail.
Therefore, this scenario tests the ability of the OVAL to detect that the system is broken, and the ability of the remediation to fix it.
Precisely, we test the ability of the remediation to satisfy the OVAL check - the test suite doesn't do functional tests, i.e. it doesn't check that the system behaves in a particular way - it just executes the scanner.
- Inside, we see that there is a line containing `# variables = var_accounts_tmout=600` - that indicates that the rule is parametrized, and we force the value of the parameter regardless of the rule's default, or of any possible profile setting.
[More metadata](https://github.com/ComplianceAsCode/content/tree/master/tests#scenarios-format) are supported.

Let's look a completely different scenario [missing.fail.sh](https://github.com/ComplianceAsCode/content/blob/master/linux_os/guide/system/bootloader-grub2/non-uefi/grub2_password/tests/missing.fail.sh) - for the [grub2_password](https://github.com/ComplianceAsCode/content/tree/master/linux_os/guide/system/bootloader-grub2/non-uefi/grub2_password) rule - let's check its contents:

- There is a `# remediation = none` metadata directive - that one is useful if the rule doesn't have a remediation.
In that case, the test suite is satisfied enough just after the expected failure occurs.
A remediation not being available is usually an exception, not an intention, and it can "disappear" as an unwanted side-effect of a metadata update, or because of a build system glitch.
Therefore, the test suite has the opt-out approach for remediations.
- Finally, there is the `. $SHARED/grub2.sh` shell sourcing.
The scenario setup code [can be shared](https://github.com/ComplianceAsCode/content/tree/master/tests#sharing-code-among-test-scenarios) to any scenario, so there is no need for code duplication.

I believe that now you know enough to start experimenting with test scenarios - keep in mind that the test suite can run them selectively by means of the `--scenarios` argument, which is handy when only one scenario causes issues..

Conclusion
----------

The blog post grew quite long, but we just scratched the surface - the test suite can do much more!

It can test profiles, when it can determine how "green" can the datastream make typically a freshly installed system.
This type of test just runs the scan with remediations if needed (i.e. no scenarios), but it can reveal conflicts between rules that are selected by the profile.
The test suite can also operate in combined mode that combines the profile test with rule tests - in that case, every rule in a profile is tested thoroughly using its test scenarios.

Although the container backend is very convenient, it's not the best representation of a system.
A virtual machine feels much closer to the bare-metal, and the test suite supports the `libvirt` backend, which leverages all those snapshots and VM control capabilities of `libvirt` to execute tests.
Setting up a VM that works well with the test suite is not trivial, and we have the [install_vm](https://github.com/ComplianceAsCode/content/blob/master/tests/install_vm.py) script that is capable to assist with that.

So there is a material for another blog posts - dear readers, let us know what would you like to read about!
It's never late to use either the [mailing list at scap-security-guide@lists.fedorahosted.org](https://lists.fedorahosted.org/admin/lists/scap-security-guide.lists.fedorahosted.org/), the [Github discussion forum](https://github.com/ComplianceAsCode/content/discussions), or the comments section of this website to provide feedback, and ask for more.

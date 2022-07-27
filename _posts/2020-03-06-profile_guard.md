---
layout: post
title: "Stay on Top of Your Profile"
categories: template
author: Matěj Týč
img: thumbnail/profile.png
---

# Stay on Top of Your Profile

## The problem: How is my profile?

Let's consider the obvious question "how do you feel about your profile?".
The answer certainly depends on the size of the profile in question - so how many rules and other settings define a profile?
That's not such an easy question to answer either, but profiles based on policies have typically up to four hundreds of rules selected.

Answering the original question about profile's state is made more tricky due to how ComplianceAsCode build system gives content authors space when specifying profiles.
ComplianceAsCode uses the YAML format for profile definitions, so profile creators often group rules by policy requirements, and add a lot of comments why a specific rule is selected by the profile.
As a result, profile definitions can end up having hundreds of lines.
When you consider the ability of profiles to extend an existing profile, it's really difficult to keep track of profile composition over time.
Ultimately, one can analyze the datastream and examine selections there. But, the build system knows what's selected and what's not. We can use the build system to show this information instead of analysing a complex XML file.

See, for example, a snippet of RHEL8's OSPP profile:

```
    ## Set Screen Lock Timeout Period to 30 Minutes or Less
    ## AC-11(a) / FMT_MOF_EXT.1
    ## We deliberately set sshd timeout to 1 minute before tmux lock timeout
    - sshd_idle_timeout_value=14_minutes
    - sshd_set_idle_timeout

    ## Disable Unauthenticated Login (such as Guest Accounts)
    ## FIA_AFL.1
    - require_singleuser_auth
    - grub2_disable_interactive_boot
    - grub2_uefi_password
    - no_empty_passwords
```

The `AC-11(a)`, `FMT_MOF_EXT.1` and `FIA_AFL.1` are identifiers of security requirements that have an important meaning for profile stakeholders.


## The solution: Build system tells you

The build system recently gained the ability to analyze profile definitions at build-time.
It produces a "compiled profile" file - a YAML file that contains final rule selection, variable assignments, and rule refinements, all of that sorted lexicographicaly.
In other words, profile extensions, blacklisting and overrides are resolved, comments are dropped, and selections ordering is normalized.

For example, if you would like to know what is the real contents of the RHEL8 STIG profile, you don't look to `rhel8/profiles/stig.profile`.
Instead, you build the RHEL8 datastream, and check out the `build/rhel8/profiles/stig.profile` build artifact.


## The improvement: Stability test

A problem that our group has been facing in the past was an unintended introduction of profile changes.
Imagine that you decide to restructure your profile definition by regrouping selections and adding a couple of comments - the resulting diff can easily grow over a hundred lines, and humans have no chance of reviewing the pull request properly, without having to perform code review that requires a lot of effort.
Imagine also a situation when a profile you build upon changes, and you don't want those changes to propagate to your profile.

Along with the introduction of compiled profiles, we have switched on an optional test - it is now possible to compare profile contents to reference contents.
This basically means that a profile tested this way is specified twice in the project - there is its author-friendly definition in `<product>/profiles/<name>.profile`, and if there may be its computer-friendly, compiled definition at `tests/data/profile_stability/<product>/<name>.profile`.

If the product is built and the test is executed, test references are compared to their corresponding compiled build artifacts.
If selections differ, the test fails, and a helpful message is displayed.
In order to make the test passing again, one can update the reference easily by overwriting it with the build artifact, and the change in the reference should have a really simple diff that is easy to review.


## Check it out

Let's try to modify a guarded profile, then build the content, and execute tests.
Be sure to start with a clean repository, so you can get rid of any profile damage easily by performing `git reset --hard`.

First of all, let's get rid of all selections in the RHEL8 profile that contain the string "partition" by executing this command in the project's root:

```
$ sed -i '/partition/d' rhel8/profiles/ospp.profile
```

Running git diff would tell us that a couple of rule selections were removed:

```
$ git diff

diff --git a/rhel8/profiles/ospp.profile b/rhel8/profiles/ospp.profile
index c67206605..50aae0833 100644
--- a/rhel8/profiles/ospp.profile
+++ b/rhel8/profiles/ospp.profile
@@ -31,17 +31,12 @@ selections:
     - mount_option_dev_shm_nodev
     - mount_option_dev_shm_noexec
     - mount_option_dev_shm_nosuid
-    - mount_option_nodev_nonroot_local_partitions
     - mount_option_boot_nodev
     - mount_option_boot_nosuid
-    - partition_for_home
-    - partition_for_var
     - mount_option_var_nodev
-    - partition_for_var_log
     - mount_option_var_log_nodev
     - mount_option_var_log_nosuid
     - mount_option_var_log_noexec
-    - partition_for_var_log_audit
     - mount_option_var_log_audit_nodev
     - mount_option_var_log_audit_nosuid
     - mount_option_var_log_audit_noexec
```

Now, let's build the RHEL8 product:

```
$ ./build_product rhel8
```

Finally, after the build finishes, let's run the test in verbose mode.
Test's name is `stable-profiles`, and we can use ctest's `-R` option to run only that single test.
The `ctest` invocation has to be executed from the build directory, so we change to it first:

```
$ cd build
$ ctest -R stable-profiles  -V
...
Checking test dependency graph...
Checking test dependency graph end
test 24
    Start 24: stable-profiles
...
24: Following selections were removed from the rhel8's stig profile:
24:  - partition_for_home
24:  - partition_for_var_log
24:  - partition_for_var
24:  - partition_for_var_log_audit
24:  - mount_option_nodev_nonroot_local_partitions
24: Following selections were removed from the rhel8's ospp profile:
24:  - partition_for_home
24:  - partition_for_var_log
24:  - partition_for_var
24:  - partition_for_var_log_audit
24:  - mount_option_nodev_nonroot_local_partitions
24: If changes to mentioned profiles are intentional, copy those compiled files, so they become the new reference:
24: cp '.../build/rhel8/profiles/stig.profile' '.../tests/data/profile_stability/rhel8/stig.profile'
24: cp '.../build/rhel8/profiles/ospp.profile' '.../tests/data/profile_stability/rhel8/ospp.profile'
24: Please remember that if you change a profile that is extended by other profiles, changes propagate to derived profiles.
If those changes are unwanted, you have to supress them using explicit selections or !unselections in derived profiles.
1/1 Test #24: stable-profiles ..................***Failed    0.21 sec

0% tests passed, 1 tests failed out of 1

Total Test time (real) =   0.21 sec

The following tests FAILED:
         24 - stable-profiles (Failed)
Errors while running CTest
```

Please note that parts of the output were skipped and substituted by ellipsis (i.e. by `...`):

If you perform suggested copy commands and re-run tests, they then pass.
However, when you submit a pull request with all files changed, there won't be any doubt that those rules were removed from the profile, even if the change is part of a mass rearranging of the `ospp.profile` file.


## Conclusion

If you are a profile developer and you want to have a strong control over your profile, consider adding the compiled profile to the test directory.
If there are legitimate updates of dependent profiles, your test reference will be overwritten, so those legitimate changes aren't blocked.
However, you can watch the history of the reference, and react to its changes - every change of an extended profile can be neutralized by adding selections, adding unselections, or overriding variables or refinements to their original values.

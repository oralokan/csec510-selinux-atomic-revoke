# csec510-selinux-atomic-revoke

## About

This repository contains files related to one of the case studies that was part of our project on SELinux for the CSEC 510 Operating System Security course.

The aim of these case studies is to demonstrate the benefits of using SELinux, in order to do things that would otherwise not be possible using traditional Unix DAC alone.

Feel free to use whatever is contained in this repository, but keep the following two things in mind:

1. The code included can be potentially destructive (i.e. it can modify or delete data on your system). It should only ever be run in dedicated VMs for created for testing purposes and not on production systems.
2. Coming up with security policies that cannot be circumvented is hard work! We make no guarantees that the sample policies included here cannot be circumvented. In any case, that is not the purpose of these case studies. We just aim to show these policies can be used to protect against very specific offending behavior, in a manner that would not be possible unless SELinux is not used.

We are not SELinux experts! We're just learning about the basics ourselves. Keep this in mind if you ever decide to use our code in any way.

## Scenario Description

This case study focuses on the atomicity of SELinux access control checks. SELinux relies on _complete mediation_. Every single system call that will access a resource is subject to access control checks. This makes SELinux highly granular, and allows it to do things that would otherwise not be possible to do with traditional Unix DAC.

The scenario is that there is a regular file that a certain user has write permissions over at a certain point of time. The user opens a file descriptor for writing. After the file descriptor is opened, the security policy is changed such that the write permission of that user to that file is revoked. We want this change in policy to be put into effect immediately, so that the very next attempt to write to the file by the now unauthorized user should fail.

## Demo Walkthrough

As a regular user, create a regular text file: `/tmp/test.txt`. Obtain a file descriptor to this file for writing by doing `cat > /tmp/test.txt`, and write a few lines. Those lines should get written to the file. Next, open up a second terminal as the same user and revoke your own write access by doing `chmod u-w /tmp/test.txt`. Under the new security policy, you should no longer be able to write anymore to the file, but this is not the case. A new call to `cat > /tmp/test.txt` will fail, but the currently running `cat` process already holds an open file descriptor to the file, and Unix DAC only checks the ACL of a file when the file descriptor is first opened (i.e. the `open` system call).

Note that the user should be non-root. The reason is because if that, even if you revoke write access for the root user, if the `uid` is 0 (i.e. root), file permission checks don't look at the ACL. In other words, `cat > /tmp/test.txt` will _not_ fail if that file belongs to root even if its ACL is 0.

### Installing the `atomicrevoke` Policy Module

The `atomicrevoke` SELinux policy module defines the `atomicrevoke_t` type. A number of permissions are granted to an unconfined user (which is the defualt for root in the case of a targeted policy on Fedora 27). However these do not include the write permission.

Calling `bin/enable-policy-module` will compile and install the policy module.

### Repeating the Experiment With The Module Enabled

Now that we have enabled the module, we repeat the steps above, but make it even harder for ourselves by working as the root user. When we decide to revoke write access, we change the file context using `chcon -t atomicrevoke_t /tmp/test.txt`. Under the new context, the root user will no longer have write permission on the file, and the very next attempt to write to the existing file descriptor will fail, which is what we want.
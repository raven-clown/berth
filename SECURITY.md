# Security policy

berth runs containers as root on Linux, so security bugs matter even before the first release.

## Reporting a vulnerability

Report vulnerabilities privately through [GitHub's private vulnerability reporting](https://github.com/raven-clown/berth/security/advisories/new). Do not open a public issue, pull request or discussion for them.

A useful report includes:

- what an attacker can do, for example escape a container, write outside a rootfs while unpacking, or reach the API without permission
- the steps or a proof of concept
- the berth version or commit, the kernel version and the distribution

You will get an answer within 7 days. Once a fix is ready, it is released together with an advisory that credits you, unless you ask not to be named.

## Supported versions

No version has been released yet. After v0.1, the latest minor release gets security fixes.

## Scope

In scope:
- berth's own code: the CLI, daemon, shim, runtime adapter, image unpacking, networking and API.

Out of scope:
- Bugs in upstream projects such as youki's libcontainer or the Linux kernel. Report those upstream. If berth's use of them makes a bug worse, report that here as well.

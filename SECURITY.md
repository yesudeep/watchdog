# Security Policy

## Supported versions

Security fixes are applied to the latest released version. Older series do not
receive backports; please upgrade before reporting an issue against them.

| Version | Supported |
| ------- | --------- |
| 7.0.x   | ✅        |
| < 7.0   | ❌        |

## Reporting a vulnerability

**Please do not report security vulnerabilities through public GitHub issues,
pull requests, or discussions.**

Report privately through GitHub's private vulnerability reporting:

<https://github.com/gorakhargosh/watchdog/security/advisories/new>

If that is unavailable to you, email the maintainer at <contact@tiger-222.fr>.

Please include as much of the following as you can:

- the affected version, operating system, and observer backend
  (inotify, FSEvents, kqueue, ReadDirectoryChangesW, or polling)
- a description of the issue and its impact
- steps to reproduce, ideally a minimal script
- any proof-of-concept code

## What to expect

- **Acknowledgement** within 7 days.
- **Initial assessment** within 14 days, including whether we consider it a
  vulnerability and a rough timeline.
- **Disclosure** coordinated with you. We will credit you in the advisory unless
  you prefer otherwise.

Fixes are published as a GitHub security advisory on this repository and noted
in [`changelog.rst`](changelog.rst).

## Scope

watchdog is a library and a command-line tool (`watchmedo`) for monitoring
filesystem events. Reports are in scope when they concern the watchdog codebase
itself — for example memory-safety issues in the macOS `_watchdog_fsevents`
extension module, or unsafe handling of filesystem metadata.

The following are generally **out of scope**:

- Vulnerabilities in dependencies. Report those to the relevant project, though
  we are glad to hear about them if watchdog's usage makes them exploitable.
- Behaviour that requires the operator to already have the ability to run
  arbitrary code as the watchdog user.

If you are unsure whether something is in scope, report it anyway.

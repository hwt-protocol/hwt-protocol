# Contributing to the HWT Protocol

## Project structure

This is one of three repositories:

- **hwt-protocol** (here) — normative specification, registry, governance documents. No implementation code.
- **hwtr-js** — JavaScript reference implementation and tests.
- **hwt-demo** — working demos and deployment baselines.

Make sure you're in the right place before opening an issue or PR.

---

## Stability posture

HWT is an intentionally stable protocol. The spec is not seeking new features. The primary contribution surfaces here are errata, registry entries, and documentation improvements. Protocol behavior changes are rare and require strong justification.

If you want to add a feature, open a Discussion first. Don't write code or a formal proposal until there's agreement that the problem is real and the scope is right.

---

## What belongs here

- Spec errata (typos, broken links, clarifications with no normative effect)
- Registry entries — new `authz` schema identifiers, jurisdiction identifiers, codecs
- Corrections to normative language that doesn't match intended behavior
- Documentation improvements

**Does not belong here:** implementation bugs, usage questions, deployment help. Those go to [hwtr-js](../hwtr-js) issues or Discussions.

For external implementations in other languages, see [IMPLEMENTATIONS.md](./IMPLEMENTATIONS.md).

---

## Issues

Use issues for confirmed errata and concrete bugs in the spec text — something that is demonstrably wrong or inconsistent. For questions, design proposals, and anything not yet clearly scoped, use [Discussions](../../discussions) instead.

---

## Discussions

Open a Discussion for:

- Questions about protocol behavior or design intent
- Proposals for protocol changes or extensions
- Registry entry proposals that need vetting before a PR
- Anything you're not sure belongs as an issue

A Discussion that reaches consensus and clear scope can become a PR. Registry entry PRs submitted without a prior Discussion are welcome if the entry is well-specified and follows the existing pattern in [CONVENTIONS.md](./CONVENTIONS.md) — otherwise, start in Discussions.

---

## Pull requests

PRs are welcome for:

- Errata fixes
- Registry entries (with or without a prior Discussion, per above)
- Documentation improvements

PRs require a sign-off on every commit (see DCO below). PRs that propose normative behavior changes without a prior Discussion will be closed with a pointer to open one.

Minor errata may be submitted as a PR directly with no prior issue or Discussion.

---

## Style

Write normative behavior using RFC 2119 keywords: **MUST**, **SHOULD**, **MAY**, **MUST NOT**, **SHOULD NOT**. Use these exactly. Avoid "required to", "expected to", "it is recommended that" in normative sections.

Implementation guidance and deployment conventions belong in [CONVENTIONS.md](./CONVENTIONS.md) or Appendix A of the spec (non-normative), not in normative sections.

---

## Developer Certificate of Origin

This project uses the [Developer Certificate of Origin (DCO)](https://developercertificate.org/). No CLA required.

Sign off each commit:

```
git commit -s -m "your commit message"
```

This adds `Signed-off-by: Your Name <your@reachable-email-address>` to the commit. PRs without sign-off will not be merged.

Full DCO text:

```
Developer Certificate of Origin
Version 1.1

Copyright (C) 2004, 2006 The Linux Foundation and its contributors.

Everyone is permitted to copy and distribute verbatim copies of this
license document, but changing it is not allowed.

Developer's Certificate of Origin 1.1

By making a contribution to this project, I certify that:

(a) The contribution was created in whole or in part by me and I
    have the right to submit it under the open source license
    indicated in the file; or

(b) The contribution is based upon previous work that, to the best
    of my knowledge, is covered under an appropriate open source
    license and I have the right under that license to submit that
    work with modifications, whether created in whole or in part
    by me, under the same open source license (unless I am
    permitted to submit under a different license), as indicated
    in the file; or

(c) The contribution was provided directly to me by some other
    person who certified (a), (b) or (c) and I have not modified
    it.

(d) I understand and agree that this project and the contribution
    are public and that a record of the contribution (including all
    personal information I submit with it, including my sign-off) is
    maintained indefinitely and may be redistributed consistent with
    this project or the open source license(s) involved.
```

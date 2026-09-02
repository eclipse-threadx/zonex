# Contributing to Eclipse ThreadX

Thanks for your interest in this project.

## Project description

Eclipse ThreadX provides a vendor-neutral, open source, safety-certified OS for real-time applications, published under a permissive license. The Eclipse ThreadX suite encompasses:

* **ThreadX** - advanced real-time operating system (RTOS) designed specifically for deeply embedded applications
* **NetX Duo** - advanced, industrial-grade TCP/IP network stack designed specifically for deeply embedded real-time and IoT applications
* **FileX** - high-performance, FAT-compatible file system that is fully integrated with the ThreadX kernel
* **GUIX** - complete, embedded graphical user interface (GUI) library
* **GUIX Studio** - design environment, facilitating the creation and maintenance of all graphical elements for GUIX
* **USBX** - high-performance USB host, device, and on-the-go (OTG) embedded stack, fully integrated with the ThreadX kernel
* **LevelX** - flash wear levelling for FileX and stand-alone purposes
* **TraceX** - analysis tool that provides a graphical view of real-time system events to better understand the behaviour of real-time systems
* **ZoneX** - deterministic partitioning hypervisor for mixed-criticality embedded systems
* **SampleX** - samples and demos for the components above

Project websites:

* https://threadx.io
* https://projects.eclipse.org/projects/iot.threadx

This file describes how to contribute to **ZoneX**. The same conventions apply across the Eclipse ThreadX repositories.

## Terms of Use

This repository is subject to the Terms of Use of the Eclipse Foundation: https://www.eclipse.org/legal/termsofuse.php

## New contributors

Welcome. Here is the shortest path from zero to a merged pull request.

1. **Create an Eclipse Foundation account.** https://accounts.eclipse.org/user/register

   The email address on that account matters: it must be the same address you use as the `Author` of your Git commits. If the two do not match, the ECA check on your pull request will fail.

2. **Sign the Eclipse Contributor Agreement (ECA).** See the section below. This is a one-time step and covers every Eclipse Foundation project.

3. **Configure Git accordingly.**

   ```
   git config user.name "Your Name"
   git config user.email "the-address-on-your-eclipse-account@example.org"
   ```

4. **Pick something to work on.** Open issues are fair game, including ones nobody has assigned. Please leave a comment on the issue saying you intend to work on it, so two people do not solve the same problem twice. If an issue carries a `good first issue` label, it is a reasonable starting point.

5. **Have an idea for a new feature? Discuss it first.** Open a thread in [GitHub Discussions](https://github.com/orgs/eclipse-threadx/discussions) or raise it on the [developer mailing list](https://accounts.eclipse.org/mailing-list/threadx-dev) before writing code. ZoneX is a hypervisor for mixed-criticality systems; design decisions there have consequences that are expensive to reverse once merged, so we would rather talk about the shape of a feature early than turn down a finished pull request.

Bug fixes and documentation corrections need no prior discussion. Send them straight in.

## Eclipse Development Process

This Eclipse Foundation open project is governed by the Eclipse Foundation Development Process and operates under the terms of the Eclipse IP Policy.

* https://eclipse.org/projects/dev_process
* https://www.eclipse.org/org/documents/Eclipse_IP_Policy.pdf

## Eclipse Contributor Agreement

In order to be able to contribute to Eclipse Foundation projects you must electronically sign the Eclipse Contributor Agreement (ECA): https://www.eclipse.org/legal/ECA.php

The ECA provides the Eclipse Foundation with a permanent record that you agree that each of your contributions will comply with the commitments documented in the Developer Certificate of Origin (DCO). Having an ECA on file associated with the email address matching the "Author" field of your contribution's Git commits fulfills the DCO's requirement that you sign-off on your contributions.

For more information, please see the Eclipse Committer Handbook: https://www.eclipse.org/projects/handbook/#resources-commit

## Required tooling

Eclipse ThreadX components build with CMake and Ninja. There is no dependency on any IDE.

**ZoneX is built and tested on Linux only.** Its targets are the Armv8-R AEM FVP and the NXP S32Z280-594EVB, both cross-compiled with the Arm toolchains, and its host unit tests are built with GCC. There is no Windows build and no PowerShell script set; see [`docs/decisions.md`](docs/decisions.md) D13, which also records how cheaply that could change if ZoneX ever acquires a reason for it. Other components of the suite do support Windows through the Visual Studio Build Tools, with deliberately no dependency on Visual Studio itself.

| Tool | Requirement |
| ---- | ----------- |
| CMake | 3.28 or later |
| Ninja | any recent release |
| Git | any recent release |
| Python | 3.x, with `gcovr` 8.6 for coverage reports |

Compilers:

| Platform | Default compiler |
| -------- | ---------------- |
| Linux (host builds and tests) | GCC 14 |
| Windows (host builds and tests) | MSVC, via the Visual Studio Build Tools |
| Arm cross builds, GNU | Arm GNU Toolchain **14.3.rel1** (`arm-none-eabi`, `aarch64-none-elf`) |
| Arm cross builds, LLVM | Arm Toolchain for Embedded (ATfE) **22.1.0** |

These versions are what CI pins and what the project treats as the reference. Building with a different version is fine while you develop, but a contribution is only considered verified once it passes with the versions above.

All assembly code targeting Linux toolchains must use GCC syntax.

## Building and testing

ZoneX follows the convention the other Eclipse ThreadX repositories use: `scripts/` holds one build script and one test script per test target, each a thin wrapper over a `run.sh` under `test/` where the logic actually lives.

| Script pair | Target | What it does |
| ----------- | ------ | ------------ |
| `scripts/build_host.sh`, `scripts/test_host.sh` | host | Builds and runs the host unit tests over the architecture-independent code. Pass `coverage` to `test_host.sh` for a gcovr report. |
| `scripts/build_fvp.sh`, `scripts/test_fvp.sh` | Armv8-R AEM FVP | Cross-builds the Cortex-R52 images and executes them on the model. |
| `scripts/build_s32z280.sh`, `scripts/test_s32z280.sh` | NXP S32Z280-594EVB | Cross-builds the same images for silicon. Running them needs the board. |
| `scripts/install.sh` | — | Installs the build and test dependencies on Ubuntu. |
| `scripts/check_terminology.sh` | — | Rejects register and concept names that belong to other architectures. See below. |

`CMakePresets.json` offers the same builds directly: `--preset default` for a warning-tolerant host build, `--preset ci-strict` for the host build with warnings as errors, `--preset coverage`, and `--preset fvp` / `--preset s32z280` for the cross builds.

**Which suite does your change belong in?** ZoneX runs two, and the split is deliberate. Architecture-independent logic — the partition manifest and its validator, the partition tables, the schedule arithmetic — is covered by the host suite, which runs anywhere in seconds. Stage-2 MPU programming, the trap path, isolation and every timing claim are only true on the FVP and on silicon and are tested there; a host simulator would be testing a simulation of the mechanism rather than the mechanism. [`docs/decisions.md`](docs/decisions.md) D11 has the full reasoning, including why ZoneX does not hold the suite's usual coverage threshold over the whole repository.

**A note on terminology.** ZoneX targets Armv8-R AArch32, where both stages of address control are region-based MPUs. Register names from the AArch64 system-register set, the translation-table registers, and RISC-V memory-protection vocabulary are wrong here by construction, and code that uses one was written against the wrong architecture. `scripts/check_terminology.sh` checks this mechanically and runs in CI; [`docs/armv8r-el2-reference.md`](docs/armv8r-el2-reference.md) holds the verified names, encodings and field layouts. Read it before writing anything that touches a register.

**ZoneX is written to C17**, not C99 — it is the one component of the suite born on that baseline. Extensions are off and `-Wpedantic` is in force, so GNU-only constructs are rejected.

Whatever you build, describe in your pull request how you verified your change. "It builds" is not verification.

## Continuous integration

Four GitHub Actions workflows. **Every one of them triggers on `pull_request` against `dev` and `main`, and on `push` to `dev` and `main`** — a workflow that gates no pull request anybody opens is worse than no workflow, because it looks like coverage.

| Workflow | What it checks |
| -------- | -------------- |
| `host_tests.yml` | The host unit tests, built with warnings as errors, plus a coverage report. Runs `check_terminology.sh` as a separate job so its answer is unambiguous. |
| `gcc_check.yml` | Cross-builds every Cortex-R52 configuration — FVP, S32Z280, hard float — with the Arm GNU Toolchain and warnings as errors. Compiles and links; executes nothing. |
| `clang_check.yml` | The same sources with Arm Toolchain for Embedded. GNU `as` accepts non-canonical assembly forms that LLVM's assembler rejects, and ZoneX is going to be substantially assembly. |
| `zx_fvp.yml` | Builds the Cortex-R52 images and **executes** them on the Armv8-R AEM FVP, judging each by its self-reported result. There is no static check for "the partition still runs". |

Arm distributes the Armv8-R AEM FVP free of charge but behind a click-through licence with no stable unauthenticated URL, so `zx_fvp.yml` takes the download location from the `FVP_AEMV8R_URL` repository variable (with an optional `FVP_AEMV8R_SHA256`). When it is unset the build lanes still run and still gate the pull request; only execution is skipped, and it says so in the log and in the job summary rather than passing quietly.

Toolchain versions are pinned explicitly, so a bump is a reviewable commit. Third-party actions are pinned to a commit SHA rather than a tag, and `.github/dependabot.yml` keeps those pins moving — a SHA pin without Dependabot freezes CI on whatever was current the day it was written.

**One thing to know if you have a pull request open already:** a workflow added after your branch was cut does not appear on it. Only a rebase makes new checks show up.

## Pull request acceptance criteria

**Pull requests must target the `dev` branch.** `main` always holds the latest release; see [Release model and support](#release-model-and-support) below. A pull request opened against `main` will be asked to retarget.

Before requesting a review, check your contribution against this list.

**Process**

* The branch is a feature branch based on `dev`. Never commit directly to `main` or `dev`.
* Your ECA is signed and the commit `Author` email matches your Eclipse account.
* The pull request is one logical change. Unrelated fixes belong in separate pull requests.
* Commit subject lines start with a past-tense verb, for example `Fixed the memory allocator`.
* The pull request explains what changed and why, and how you verified it.

**Code**

* The code is C17-compatible, and compiles clean with extensions off and `-Wpedantic`. (ZoneX differs from the rest of the suite here; see [`docs/decisions.md`](docs/decisions.md) D12.)
* It follows the coding style of the surrounding code.
* New functions and structures are documented in comments, as in existing code.
* MISRA C rules are followed as closely as practical, taking MISRA C 2004, 2012 and 2023 into account. Any deviation is explicit, names the rule being circumvented, and justifies it in a comment.
* `goto` is not used.
* No new external dependencies. This is a hard rule.
* When implementing an industry standard, code is not copied from an existing implementation. Existing implementations may inform your work, but you must identify those sources clearly.
* Code is written with ThreadX's priorities in mind: runtime speed and small code size.
* New and modified files carry the correct copyright header (see below).

**Verification**

* All CI checks are green.
* The change builds without new warnings on the reference toolchains.
* Regression tests covering the change are added or updated. The project targets 100% test coverage; a pull request that lowers coverage needs a stated reason.
* API or behaviour changes come with a matching documentation pull request against [rtos-docs-asciidoc](https://github.com/eclipse-threadx/rtos-docs-asciidoc).

**Security**

If you discover a security issue while working on a change, do not describe it in a public pull request. Follow [SECURITY.md](SECURITY.md) instead.

## AI-assisted contributions

**AI-assisted contributions are welcome**, provided they are attributed.

Two things are non-negotiable:

1. **Attribution.** Say which tool produced the code, in the file header and in the commit message. The sections below give the exact form.
2. **Human responsibility.** The human contributor submitting the pull request is responsible for the contribution - technically and legally. Signing the ECA means *you* certify the contribution's provenance. An AI tool cannot sign the ECA and cannot hold that responsibility. Review what the tool produced, understand it, and verify that it is correct and that you have the right to contribute it. "The model wrote it" is not a defence.

This is consistent with the Eclipse Foundation's [Generative AI Usage Guidelines](https://www.eclipse.org/projects/guidelines/genai/) and the [Eclipse Project Handbook](https://www.eclipse.org/projects/handbook/#genai). Please read them before submitting AI-assisted work.

### Using a coding agent

There is no `AGENTS.md` in this repository, and no other Eclipse ThreadX repository carries one either. This document is the authority, and it is what to point an agent at. A per-repository restatement of these rules would be one more place for them to drift out of step with the document that governs them.

Most agents read a file named `AGENTS.md` or `CLAUDE.md` from the working tree automatically. If you keep one, keep it untracked.

What an agent working on ZoneX has to be told, all of it either covered below or linked from here:

* **What is different about ZoneX** - C17 rather than C99, the `zx_` / `ZX_` prefix, and the Armv8-R AArch32 terminology rules that `scripts/check_terminology.sh` enforces in CI. See [`docs/decisions.md`](docs/decisions.md) D12 and D1, and [`docs/armv8r-el2-reference.md`](docs/armv8r-el2-reference.md) for the verified register names.
* **General and style rules** - follow the surrounding code, MISRA C, no `goto`, document new functions and structures, optimise for speed, code size and predictable worst-case timing.
* **Compilers** - Arm GNU Toolchain 14.3.rel1 and ATfE 22.1.0 for cross builds, GCC 14 for the host build, GNU assembly syntax, CMake and Ninja for everything.
* **Headers** - the copyright and AI-disclosure templates below. Every file in this repository is new, so the new-file form is the only one ZoneX needs.
* **Dependencies** - external dependencies are forbidden; never copy code from another implementation of a standard.
* **Regression tests** - 100% coverage, tests ship with the feature, and the change goes in whichever of the two suites can actually exercise it. See "Building and testing" above.
* **Git** - feature branches based on `dev`, never commit to `main` or `dev` directly, past-tense commit subjects, `Assisted-by` attribution and never `Co-Authored-By`.
* **Security advisories** - draft a GHSA rather than describing the issue in a public pull request.
* **Documentation** - manual changes belong in `rtos-docs-asciidoc`; design decisions belong in [`docs/decisions.md`](docs/decisions.md) and verified hardware facts in [`docs/armv8r-el2-reference.md`](docs/armv8r-el2-reference.md).

Whatever the agent produced, you are the contributor. The checklist under "Pull request acceptance criteria" above applies unchanged.

### Header for new files

Add this header when creating a new C or assembly (`.S`) file:

```c
/***************************************************************************
 * Copyright (c) <current_year> Eclipse ThreadX contributors
 *
 * This program and the accompanying materials are made available under the
 * terms of the MIT License which is available at
 * https://opensource.org/licenses/MIT.
 *
 * AI Disclosure: This file was largely AI-generated by <product> (<model_and_version>).
 * The AI-generated portions may be considered public domain (CC0-1.0)
 * and not subject to the project's licence. The human contributor has
 * reviewed and verified that the code is correct.
 *
 * SPDX-License-Identifier: MIT and CC0-1.0
 **************************************************************************/
```

Substitute the current year, the product name, and the model and version. Concretely:

```c
/***************************************************************************
 * Copyright (c) 2026 Eclipse ThreadX contributors
 *
 * This program and the accompanying materials are made available under the
 * terms of the MIT License which is available at
 * https://opensource.org/licenses/MIT.
 *
 * AI Disclosure: This file was largely AI-generated by Claude Code (Opus 5).
 * The AI-generated portions may be considered public domain (CC0-1.0)
 * and not subject to the project's licence. The human contributor has
 * reviewed and verified that the code is correct.
 *
 * SPDX-License-Identifier: MIT and CC0-1.0
 **************************************************************************/
```

Other examples of `<product> (<model_and_version>)`: `Codex (GPT-5.4)`, `Copilot (Sonnet 4.6)`, `Gemini CLI (Gemini 3 Pro)`.

If a file was written by a human without AI assistance, drop the AI Disclosure paragraph and use `SPDX-License-Identifier: MIT`.

### Header for existing files

When editing an existing C or assembly file whose copyright is from 2025 or earlier, and which does not already mention Eclipse ThreadX contributors, add this line to the header:

```c
 * Copyright (c) <current_year> Eclipse ThreadX contributors
```

It goes *below* the older copyright. All copyright lines must stay in chronological order. For example:

```c
/***************************************************************************
 * Copyright (c) 2024 Microsoft Corporation
 * Copyright (c) 2026 Eclipse ThreadX contributors
 *
 * This program and the accompanying materials are made available under the
 * terms of the MIT License which is available at
 * https://opensource.org/licenses/MIT.
 *
 * SPDX-License-Identifier: MIT
 **************************************************************************/
```

If the edit was AI-assisted, add this line just under the header:

```c
// Some portions generated by Claude Code (Opus 5).
```

If you copy an existing file to get started on a new one, treat the result as a new file: use the new-file header, do not carry the old one over.

### Commit attribution

Attribute AI assistance in the commit message with an `Assisted-by` trailer:

```
Fixed the guest context switch on AArch64

The saved SPSR was restored before the general-purpose registers, so a
guest resuming from an exception observed the wrong PSTATE.

Assisted-by: Claude Code (Opus 5) <noreply@anthropic.com>
```

## Release model and support

This section summarises the project's [Release Model and Support Policy](https://github.com/eclipse-threadx/rtos-docs-asciidoc/blob/main/rtos-docs/home/modules/ROOT/pages/releases-and-support.adoc), which is the authoritative version.

### Version numbers

Eclipse ThreadX releases generally follow [Semantic Versioning](https://semver.org/). Given a version number **X.Y.Z.Bh**, for example `6.5.0.202601a`:

* **X** increases for a *milestone release*. Components currently stay at version 6.
* **Y** increases for a *feature release* adding a major feature.
* **Z** increases for a *maintenance release* of minor fixes and improvements.
* **B** is a *build number* identifying the quarter of publication - `202601` is Q1 2026.
* **h** denotes a *hotfix release*, identified by a letter. Hotfix releases are component-specific.

### Release cadence

The project adopted a predictable quarterly release model in September 2025 and has published a release every quarter since.

Quarterly releases ship new versions of every component, whether or not that component saw code changes, so that version numbers do not drift apart between components. A release shipping no code changes for a component says so in its release notes.

Urgent fixes between quarterly releases ship as *hotfix* releases. Those contain only security fixes, or fixes for serious problems that prevent building or testing applications.

### Branches

The project runs a time-based release train on trunk-based development. There are no long-term support branches and no backports to older releases.

* `main` (or `master`, in the older repositories) always contains the latest quarterly or hotfix release.
* `dev` is the integration branch. Pull requests are merged there during the quarter. Track `dev` if you want to test new features and fixes ahead of a release.
* A quarterly release is performed by merging `dev` into `main`.

### Support policy

Issues in the codebase are fixed on a best-effort basis, and pull requests fixing existing or new issues are gladly accepted.

Security vulnerabilities disclosed confidentially are handled under the [Eclipse Foundation's security policy](https://www.eclipse.org/security/policy/). Once a candidate vulnerability is confirmed, the team works to deliver a fix as soon as possible, shipping it in the next quarterly release or as a hotfix depending on timing. Resolved vulnerabilities are disclosed immediately after a release containing the fix becomes available. The team aims to resolve newly confirmed vulnerabilities within three months; that period may be extended by the Project Leadership Chain together with the Eclipse Foundation Security team where appropriate.

See [SECURITY.md](SECURITY.md) for how to report a vulnerability.

### Roadmap

The project team plans its work on a [public GitHub project board](https://github.com/orgs/eclipse-threadx/projects/2/views/2).

### A note on older releases

Microsoft contributed only the Azure RTOS codebase v6.x to the Eclipse Foundation. Older ThreadX releases (v5.x and lower) sold by Express Logic were never made open source, and Microsoft has discontinued sales and support for them. Users of ThreadX 5.x and lower should upgrade to the latest release of Eclipse ThreadX as soon as possible.

## Documentation

The documentation is published at **https://threadx.io**. PDF manuals for every component, in A4 and US Letter formats, are attached as assets to each GitHub release.

### The stack

Documentation is written in [AsciiDoc](https://asciidoc.org/) and built with [Antora](https://antora.org/). PDF manuals are produced by the [Antora Assembler](https://docs.antora.org/assembler/latest/) with `asciidoctor-pdf`.

### The workflow

1. The single source of truth is [rtos-docs-asciidoc](https://github.com/eclipse-threadx/rtos-docs-asciidoc). All documentation changes are made there, as pull requests, following the same branch rules as the code repositories.
2. At release time the site is generated from that source into [rtos-docs-html](https://github.com/eclipse-threadx/rtos-docs-html), which holds the rendered HTML and exists solely for website integration. Do not edit it by hand - your changes will be overwritten by the next build.
3. The website then serves the generated content from `rtos-docs-html`.

The older `rtos-docs` repository, which held the documentation in Markdown, is archived and superseded by `rtos-docs-asciidoc`. Do not send changes there.

If your contribution adds or changes an API or a feature, open a matching pull request against `rtos-docs-asciidoc`.

## Developer resources

Information regarding source code management, builds, coding standards, and more: https://projects.eclipse.org/projects/iot.threadx/developer

The project maintains the following repositories:

**Components**

* https://github.com/eclipse-threadx/threadx
* https://github.com/eclipse-threadx/netxduo
* https://github.com/eclipse-threadx/filex
* https://github.com/eclipse-threadx/guix
* https://github.com/eclipse-threadx/usbx
* https://github.com/eclipse-threadx/levelx
* https://github.com/eclipse-threadx/tracex
* https://github.com/eclipse-threadx/zonex

**Samples and platforms**

* https://github.com/eclipse-threadx/samplex
* https://github.com/eclipse-threadx/supported-platforms

**Documentation**

* https://github.com/eclipse-threadx/rtos-docs-asciidoc
* https://github.com/eclipse-threadx/rtos-docs-html

**Community and process**

* https://github.com/eclipse-threadx/discussions
* https://github.com/eclipse-threadx/trustedx
* https://github.com/eclipse-threadx/.github

## Contact

### GitHub Discussions

https://github.com/orgs/eclipse-threadx/discussions

Q&A, feedback, and announcements. Decisions taken by the project team are documented here as well. This is usually the fastest way to reach both the team and other users.

### Main ThreadX mailing list

https://accounts.eclipse.org/mailing-list/threadx

News and updates about the ThreadX project and the ThreadX Alliance.

### Developer mailing list

https://accounts.eclipse.org/mailing-list/threadx-dev

Project team conversations. Feel free to jump in and ask non-technical questions there.

### User mailing list

https://accounts.eclipse.org/mailing-list/threadx-users

Ask your technical questions and discuss issues here.

---
marp: true
title: How Nix defends against supply chain attacks
---

# How Nix defends against supply chain attacks

## The Problem: Supply-Chain Attack

Malicious code injected via dependencies, build tools, or package registries.

- traditional package managers trust remote servers, mutable state, and post-install scripts
- example: `event-stream` npm compromise (2018) shipped malicious dependency to steal wallet credentials from downstream apps
- Nix doesn't make malicious source impossible, but controls when and how it is run and reduces the blast radius

---

# How Nix defends

Nix is a functional, immutable system. Most protections are consequences of this core design, not bolt-on mitigations.

- pure evaluation
- sandboxed builds
- immutable store
- hash-addressed inputs and locked flakes
- no ambient mutable state or post-install mutation

---

## Pure Evaluation

- Nix expressions evaluate without network or filesystem side effects
- dependency graph becomes explicit before build starts

---

## Build Sandbox

- build runs with declared inputs only
- no ambient `/usr`, user home, credentials, or network
- compiler/toolchain dependencies come from derivation closure
- network access only possible when ouput hash is known before build

Only effect of build is: output directory created.

---

## Immutable Store

- packages live under `/nix/store/<hash>-name`
- path includes hash of all inputs (deps, build script, build options)
- store paths are immutable after build
- multiple versions coexist without overwriting each other (hash is different)
- rollback works because old closures stay addressable

All packages live in immutable, hash-addressed store.
 
---

## No Mutable State

- packages are built first, then referenced by profile or system generation
- install does not execute arbitrary package hooks on target machine like `npm postinstall`
- build-time scripts run in sandbox
- system activation is declarative and reviewable

Mutation moves from user machines to reproducible build plans.

---

## Binary Cache

- binary substituters serve prebuilt store paths
- Nix verifies signatures from trusted public keys
- content hashes bind output identity
- cache compromise alone should not bypass signature verification

Trust becomes explicit: which cache keys are trusted, which inputs are locked, which derivations produce outputs.

---

## Input locking

- all nix builds have hash locked inputs.
- ebuilds use same inputs, unless hash is changed in pacakge definition
- flakes manage input hashes in a lockfile

No part of your software relies on pre-installed stuff on build or run machine.  *Every* dependency change requires lockfile change.

---

## Be vigilent! Other attack vectors are many.

- malicious upstream source
- compromised `nixpkgs` maintainer or review path
- random flake from GitHub with dangerous build logic
- trusted binary cache key compromised
- developer disables sandbox or trusts wrong substituter

"I ran a program from the internet" is kind of an invalid threat model so there are limits to any approach.

---

## Mitigations and inspection

```sh
nix why-depends nixpkgs#firefox nixpkgs#glibc
nix path-info -r nixpkgs#firefox
nix run nixpkgs#vulnix -- --system
```

- audit dependency reasons and closure size
- scan known CVEs with `vulnix`
- use Trustix-style build transparency and reproducibility checks
- pin inputs, review lockfile diffs, minimize trusted caches

---

## How Nix defends developer laptops

- `nix develop` provides project toolchains without global installs
- package install does not run arbitrary `postinstall` scripts on developer machine
- sandbox blocks builds from reading home directory secrets by default
- pinned inputs prevent silent dependency drift between team members
- instant atomic rollback

Developer laptop becomes less special: project state comes from declared closure, not accumulated global tools.

---

## How Nix defends build machines

- CI can build from clean derivation graph instead of mutable worker image
- sandbox removes ambient credentials, `/usr`, network, and undeclared tools
- fixed-output fetchers require expected hashes for network-fetched sources
- build logs and derivations expose exact commands and dependency closure
- remote builders can reproduce same locked inputs independently

Build machine compromise gets less leverage when build has narrow inputs and no hidden environment.

---

## How Nix defends production servers

- deploy: atomic update to new immutable system
- instant atomic rollback to known good state
- runtime closure contains only declared dependencies
- binary cache signatures verify substituted artifacts before activation

Production server consumes verified closures instead of mutating itself during deploy.

---

## Summary: How Nix defends against supply chain attacks

- Nix shifts trust from runtime mutation to build-time verification
- evaluation is pure; builds are sandboxed; store is immutable
- flakes lock inputs; binary cache outputs are signature-verified
- most supply-chain defenses come free from core design
- mitigations still needed: auditability, transparency, CVE scanning, key hygiene
- keep threat model visible

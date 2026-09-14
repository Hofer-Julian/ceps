# CEP XXXX - Virtual package detection plugins

<table>
<tr><td> Title </td><td> Virtual package detection plugins </td></tr>
<tr><td> Status </td><td> Draft </td></tr>
<tr><td> Author(s) </td><td> Wolf Vollprecht &lt;wolf@prefix.dev&gt;<br/>Tobias Hunger &lt;tobias@prefix.dev&gt;</td></tr>
<tr><td> Created </td><td> Aug 5, 2026</td></tr>
<tr><td> Updated </td><td> Sep 14, 2026</td></tr>
<tr><td> Discussion </td><td> https://github.com/conda/ceps/pull/188 </td></tr>
<tr><td> Implementation </td><td> https://github.com/conda/rattler/pull/2701 (stack) </td></tr>
<tr><td> Requires </td><td> CEP 26, CEP 29, CEP 30, CEP 32, CEP 33, CEP 34, CEP 46 </td></tr>
</table>

> The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT",
> "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as
> described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) when, and only when, they appear in all capitals, as shown here.

## Abstract

[CEP 30](./cep-0030.md) standardizes a fixed set of virtual packages and makes detecting them an obligation of the client.
A client can therefore only offer the names it was written to know about, and a package that depends on any other capability of the system has no way to have that capability detected.

This CEP defines a *detection plugin*: an ordinary conda package whose executable, when run by a client, reports virtual packages as a JSON document on its standard output.
It specifies how a client installs such a plugin into an isolated environment, how it runs it, how it bounds the run, how it reads and validates the report, how the reported values enter the solve, and how they are cached and overridden.

The CEP is language-neutral: a plugin is an executable, and any client that can install a conda package and start a process can run one.
It does not say where a client learns *which* plugins to run.
That is the job of a *registration source*, and [CEP XXXX (Channel-provided virtual package plugins)](./cep-XXXX.md) defines the first one: a channel registering plugins in its `repodata.json`.

## Motivation

The set of virtual packages in CEP 30 is closed, and extending it requires a CEP per name (as [CEP 46](./cep-0046.md) did for `__cuda_arch`).
That is the right process for capabilities the whole ecosystem shares, and the wrong one for everything else:

- **Accelerator and interconnect stacks.** ROCm, oneAPI, Metal, Infiniband, TPUs and NPUs each need version *and* capability detection.
  Each is relevant to the channels shipping builds against it and to nobody else.
- **Site-specific capability.** An organization's internal channel may need to know about a license server, a filesystem, a kernel module or a CPU feature that no public channel cares about.
- **Capabilities that change faster than clients release.** A new GPU generation is a new value for an existing name.
  Today every client has to ship a new release to know about it, even though the party that builds for it knows already.

conda ships a plugin hook for virtual packages on the plugin manager [CEP 4](./cep-0004.md) introduced.
It is a Python interface, discovered through entry points of the environment that runs conda, so it cannot be used by mamba, rattler or pixi, and a plugin written for it has to be installed next to the client rather than resolved like any other package.
The conda-forge external MPI detector is the standing example: it exists, it works, and it works in exactly one client.

What is missing is a client-independent definition of what a detection plugin is, how it is run, and what it has to say.
Given that, any registration source can point at one, and every client reaches the same verdict on the same machine.

## Specification

The terms conda channel, and channel subdirectory (subdir) MUST be understood as specified by [CEP 26](./cep-0026.md).
Virtual packages are defined by [CEP 30](./cep-0030.md).
Version strings are defined by [CEP 33](./cep-0033.md).

### Terminology

- A **plugin** is a conda package that satisfies [The plugin package](#the-plugin-package).
- A **registration** tells a client to run one plugin.
  It consists of:
  - the **origin**: an identifier for where the registration came from, whose form the registration source defines (for a channel, its base URL);
  - the **plugin name**: the name of the plugin package, which is also the name of the executable to run;
  - the **declared names**: a non-empty set of valid virtual package names the plugin answers for;
  - the **resolution channels**: an ordered, non-empty list of channels the plugin and its dependencies are resolved from.

  The pair of origin and plugin name identifies a registration, and is what a user refers to when disabling or pinning one.
- A **registration source** is a specification that produces registrations.
  This CEP defines none.
  A specification defining a registration source MUST guarantee that, within one solve, at most one registration carries any given declared name, and MUST say how a client reaches that state when the source's data does not provide it.
  It MUST also say whose consent a registration carries; see [Security considerations](#security-considerations).
- The **registrant** is the party that made a registration.
- A **verdict** is what a plugin says about one declared name: present with a version and build string, or absent.
- The **closure** of a registration is the set of package records the client resolved for the plugin, as described in [Resolution](#resolution).
  The **detector environment** is the conda environment those records are installed into, and its **digest** is the identifier defined in [Identity](#identity).
- The **host platform** is the platform of the machine the client runs on, what CEP 30 calls the native platform.
  The **target platform** is the platform a solve is for.
  They are usually the same and need not be.
- A declared name is **wanted** in a solve unless an [override](#overrides) supplies it, or the registration source has assigned it to a different registration.
  A registration is **live** in a solve when it is not disabled by the user (see [What a client tells the user](#what-a-client-tells-the-user)) and at least one of its declared names is wanted.
  Only live registrations are run.

### Virtual package names

A declared name MUST satisfy the package name rules of CEP 26 and MUST begin with two underscores.
Concretely it MUST match:

```re
^__[a-z0-9][._-]?([a-z0-9]+(\.|-|_|$))*$
```

and MUST NOT exceed 64 characters.

The **override variable** of a declared name is `CONDA_OVERRIDE_` followed by the name without its two leading underscores, uppercased, with every `-` and `.` replaced by `_`.
`__acme-rocm` and `__acme.rocm` therefore both map to `CONDA_OVERRIDE_ACME_ROCM`.
A registration source MUST NOT hand a client two declared names, in one solve, that map to the same override variable, and MUST say what it does when its data contains such a pair.

A registration source MUST NOT hand a client a declared name that violates this section.
What it does with such a name is the registration source's to define.

### The plugin package

A plugin is an ordinary conda package.

- It MUST contain an executable whose file name is the plugin name in its normalized (lowercase) form, in one of the directories that [CEP 32](./cep-0032.md) puts on `PATH` when the environment is activated.
  On Windows the file MUST carry an `.exe`, `.cmd` or `.bat` extension.
- It MAY have dependencies.
- It MUST NOT depend, directly or through its dependencies, on virtual packages other than those CEP 30 and its extensions oblige the client to provide.
  A plugin is what makes other virtual packages exist, so it cannot require them.
- It and its dependencies MUST NOT rely on pre-link, post-link or pre-unlink scripts (CEP 34), because a client does not run them in a detector environment (see [Installation](#installation)).
- It SHOULD have as few dependencies as it can, and SHOULD pin the ones it has, because every dependency is code the user runs and code that can change the digest; see [Security considerations](#security-considerations).

The plugin MUST be resolvable from the registration's resolution channels for the host platform.
A plugin that is not is a failed plugin (see [Failure](#failure)), not a broken registration: a client cannot tell the two apart until it tries.

### The detector environment

#### Resolution

A client MUST resolve the plugin by a [CEP 29](./cep-0029.md) MatchSpec consisting of the plugin name and nothing else, except that a registration source MAY require the MatchSpec to carry a channel qualifier naming one of the resolution channels.
It MUST resolve against the registration's resolution channels in the given order, for the host platform, loading the `noarch` subdir alongside the host subdir.
The solver's ordinary preference for the newest version therefore decides which build of the plugin runs.
A registration source MUST NOT add version or build constraints to that MatchSpec; a registrant that wants a different build has to serve a different build.

A client MUST NOT resolve the plugin or its dependencies from any channel that is not among the resolution channels.
Which channels those are is the registration source's decision, and the point of the rule is that whoever registers a plugin also bounds the code it can pull in.

The virtual packages available to that resolution are the client's own CEP 30 virtual packages, honoring `CONDA_OVERRIDE_*`, and nothing a plugin reported.

Resolution is expected to be cheap: it runs against repodata the client has already loaded for the solve, and downloads nothing.
What the client avoids by caching is installation and running, not resolution; see [Re-resolution](#re-resolution).

#### Installation

A client MUST install the closure into an environment that is used for nothing else, following the environment layout of [CEP 32](./cep-0032.md) and the package layout of [CEP 34](./cep-0034.md).

- It MUST NOT install a plugin into the environment being solved for, and MUST NOT make a plugin a dependency of it.
  The plugin's purpose is to inform the solve, and a solve that had to contain its own detection tooling would be circular.
- It MUST NOT install a plugin into an environment the client itself runs from, or into any environment a user works in.
  Those environments change for unrelated reasons, and the digest of a detector environment is only useful as an identity if nothing but the plugin and its dependencies can change it.
- Notwithstanding CEP 32 and CEP 34, it MUST NOT execute pre-link, post-link or pre-unlink scripts of any package in the closure.
- It MAY skip the byte compilation of `noarch: python` packages ([CEP 20](./cep-0020.md)), which runs the closure's own Python interpreter over the closure's own files at install time.
  A client that performs it SHOULD bound it as it bounds activation.
  With that, the code a user runs by trusting a registration is the executable, the libraries it loads, the activation scripts of the closure, and at most that byte compilation.
- It MAY share one environment between registrations whose closures are identical, since the digest is the same.
- It MUST NOT run a plugin from an environment whose installation did not complete, and MUST make sure two concurrent installations of the same digest do not corrupt each other.

This CEP does not say where on disk a detector environment lives.

#### Identity

The digest of a closure is computed as follows.

1. For every package record in the closure, form the line `<name>\t<version>\t<build>\t<artifact>`, where:
   - `<name>` is the record's package name in normalized (lowercase) form;
   - `<version>` and `<build>` are the record's `version` and `build` strings verbatim;
   - `<artifact>` is the record's `sha256` in lowercase hexadecimal if the record carries a non-null one, otherwise its `md5` in lowercase hexadecimal if it carries a non-null one, otherwise the record's file name (`fn`).
2. Sort the lines bytewise ascending.
3. Join them with `\n`, without a trailing newline, encode as UTF-8, and take the SHA-256, rendered as lowercase hexadecimal.

CEP 26 rules out tabs and newlines in every field used.
Two artifacts of one build in different formats (`.tar.bz2` and `.conda`) have different hashes and therefore different digests, which is intended: they are different files.

The digest names the closure, and the closure is what runs.
It is the identity a client uses for caching, for reporting, and for anything a user pins or approves; see [Security considerations](#security-considerations).

#### Re-resolution

A client MUST NOT run a plugin from a detector environment it looked up by registration alone.
Before running a plugin, and before consulting its verdict cache for that registration, a client MUST resolve the registration against current repodata and compute the digest.
It MAY then serve a cached verdict whose key matches (see [Caching](#caching)), and otherwise MUST reuse an existing environment only if the digest matches, installing a new one if not.

This is what makes a plugin update take effect: the registrant publishes a new build, the next resolution produces a new closure, and the verdicts of the old one are never reused for it.

### Running a plugin

To obtain verdicts from a live registration, a client:

1. MUST compute the activated environment by evaluating the environment's activation scripts as it would for any conda environment, so that a plugin relying on activation scripts, `LD_LIBRARY_PATH`, or similar behaves as it would when used normally.
   This executes the activation scripts of every package in the closure; see [Security considerations](#security-considerations).
   Output of the activation step MUST NOT be read as part of the report.
   A failed activation is a plugin failure.
2. MUST place the environment's `PATH` directories, as CEP 32 orders them, ahead of the inherited `PATH`, so the plugin finds its own helpers before anything on the host.
3. MUST execute the executable whose file name is the normalized plugin name, looked up in those directories in that order.
   On Windows it MUST look for the `.exe`, `.cmd` and `.bat` extensions, in that order within a directory.
   If no such executable exists, the plugin has failed.
   The executable MUST be started with no arguments, with nothing on standard input, and with the activated environment; the working directory is unspecified.
4. MUST capture standard output, which carries the report, separately from standard error, which carries diagnostics.
   Both streams count against the same output bound.

A plugin MUST exit with status `0` on success.
A non-zero exit is a plugin failure.
A plugin that exits `0` having written nothing but whitespace to standard output has also failed: writing the report to standard error by mistake is common enough to be worth naming rather than reading as "nothing detected".

#### Bounds

A client MUST bound a plugin run.

- **Time.** A client MUST apply a timeout to the detector process and MUST terminate a process that exceeds it.
  It SHOULD also terminate the process's descendants, since a helper holding the output pipe open is the common way a detector outlives its timeout.
  The clock starts when the process is spawned and covers nothing before it: neither resolution, nor installation, nor activation.
  The timeout SHOULD default to **30 seconds**.
  A client MAY let a user raise it, but MUST NOT allow any timeout longer than **300 seconds**: detection happens before a solve can begin, and an unbounded plugin is an unbounded hang in a tool a user is waiting on.
  The activation step MUST be bounded separately, with the same default and ceiling, and a timed-out activation is a plugin failure.
- **Output.** A client MUST stop reading once **1 MiB** (1,048,576 bytes) of standard output and standard error combined has been read, MUST terminate the process if it is still running, and MUST treat the plugin as failed.
  The bound is the same for every plugin.
  A valid report is small, see [Size limits](#size-limits), so what the bound really limits is diagnostics, and a plugin that needs more than a megabyte of them is not diagnosing, it is logging.

Diagnostics a plugin wrote to standard error before a bound was exceeded MUST be kept and reported when the run fails.
This is to enable users to diagnose and report issues they run into, while still preventing an unbounded diagnostic stream from exhausting client memory.

#### Failure

A plugin has failed when the plugin cannot be resolved or installed, activation fails or times out, the executable is missing, the run exceeds a bound, the process exits non-zero, the report is empty or malformed, the plugin violates [the contract](#the-contract), or the registration source declares the registration failed (for example a pinned digest that does not match).

- A failed plugin MUST NOT abort the solve on its own.
- A failed plugin's verdicts MUST NOT enter the solve, including verdicts for names it did report correctly.
  Every declared name of the registration is absent for that solve, unless an [override](#overrides) supplies it or CEP 30 obliges the client to provide it, in which case the client's own value stands (see [Interaction with client-detected virtual packages](#interaction-with-client-detected-virtual-packages)).
  The solver then reports an unsatisfiable dependency on an absent name in the ordinary way.
- A client MUST report every plugin failure to the user, together with the diagnostics it captured, whether or not the solve succeeds.
  A solve that succeeds because nothing needed the failed plugin's names is still a solve in which a plugin failed, and a user who cannot see that will not understand the next solve that fails.

### The report

A plugin MUST write exactly one JSON object, encoded as UTF-8, to standard output, and nothing else but whitespace around it.
Anything else, including a top-level array, more than one value, or trailing text, is a malformed report.

A plugin registered for `__cuda`, `__cuda_arch` and `__cuda_mps` reports:

```json
{
  "version": 1,
  "virtual_packages": {
    "__cuda": { "version": "12.4" },
    "__cuda_arch": { "version": "8.9", "build_string": "0" },
    "__cuda_mps": null
  },
  "cache": {
    "ttl_seconds": 86400,
    "watch_paths": ["/sys/module/nvidia/version"],
    "watch_env": ["CUDA_VISIBLE_DEVICES"]
  }
}
```

- `version: int`.
  Required.
  The version of the report format.
  MUST be the integer `1` at this time.
  A report without it, or carrying a version the client does not implement, is a failure of that plugin: the client cannot know what the remaining keys mean.
- `virtual_packages: dict[str, dict | null]`.
  Required.
  One entry per virtual package the plugin gives a verdict about, keyed by name; keys are compared in normalized form.
  - A **dictionary** value means the virtual package is present.
    It MUST contain `version: str`, a version string as defined by CEP 33, and MAY contain `build_string: str`, which MUST satisfy the build string rules of CEP 26.
    An absent `build_string` means `0`, the default CEP 30 gives virtual packages.
    Where a name's information belongs in the version and where in the build string is decided by the CEP standardizing that name; for a name no CEP standardizes, it is the registrant's choice, and a registrant SHOULD put a version in the version so that ordinary version constraints work on it.
  - A **`null`** value means the virtual package is **not present on this system**.
    This is a verdict, and distinct from saying nothing at all.
- `cache: dict`.
  Optional.
  See [Caching](#caching).

Keys other than these MUST be ignored, at both the top level and inside a `virtual_packages` entry.
A plugin written against a later revision of this protocol then remains usable for the part the client understands, and rejecting unknown keys would buy no safety: the plugin is arbitrary code the client has already run.
A *known* key whose value has the wrong type, including a `ttl_seconds` that is negative or not an integer, makes the report malformed.

Because the report is keyed by name, a duplicate verdict cannot be expressed, and clients therefore need no rule for one.

#### Size limits

A report MUST satisfy the following, measured on the decoded string values, and a report that does not is malformed:

- `version` and `build_string` of a verdict: at most 256 bytes each.
- `watch_paths` and `watch_env`: at most 32 entries each, every entry at most 4096 bytes.

The [output bound](#bounds) applies regardless: a report that satisfies these limits but, through escaping or unknown keys, exceeds the bound has still failed.

### The contract

The registration is a promise in both directions, and a client MUST check it before anything reaches the solver:

- A plugin MUST give a verdict for **every** declared name of its registration.
  A declared name absent from `virtual_packages` is a **contract violation**, not an absent capability; that is what `null` is for.
- A plugin MUST NOT report a name that is not among its declared names.
  Doing so is a contract violation, and the client MUST NOT let the undeclared name reach the solve.

A contract violation is a plugin failure, with the consequences of [Failure](#failure): all of the plugin's verdicts are discarded, and the client reports it.
The declared names are the only part of the arrangement a user can inspect *before* code runs, and a plugin that could report anything would make them worthless.

A client MUST hold a plugin to the full contract even when only some of its declared names are wanted, and MUST discard the verdicts for names that are not wanted afterwards.
The contract is between a plugin and its registration, and the plugin cannot know what the client did with other registrations or what the user overrode.

### Verdicts in the solve

A registration source guarantees that a name is answered by at most one registration in a solve, so a verdict is a single value for a single name, and it enters the solve exactly as a client-detected virtual package does:

- A **dictionary** verdict contributes one virtual package record, with that name, version and build string, to the candidate pool.
- A **`null`** verdict contributes no record, and the name is absent for the whole solve.

Nothing else changes.

A MatchSpec on a plugin-provided name is matched by name against that one record, as CEP 29 already specifies: one candidate per name, ordinary matching, no change to MatchSpec and no new capability required of any solver.
A spec the user wrote on a command line or in a manifest needs no special treatment either, because there is nothing for it to be ambiguous between.

A record from any channel sees the same verdict, whether or not that channel had anything to do with the registration.
A registrant that wants a name only some packages use has to make the *name* distinctive; a verdict is never scoped.

### Interaction with client-detected virtual packages

CEP 30 makes the standardized virtual packages an obligation of the **client**, and for several of them dictates the value the client sets.
This CEP amends that in one respect, and leaves the rest untouched:

- **A plugin verdict for a standardized name MAY replace the client's own value.**
  Where CEP 30 or a later CEP requires a client to set a standardized name to a detected value, a client implementing this CEP MUST use a plugin's dictionary verdict for that name instead, when a live registration produced one.
  A registrant that ships a better detector for a shared capability is thereby allowed to say so.
- **A plugin cannot make a standardized name disappear.**
  A client MUST continue to provide every name CEP 30 and its extensions oblige it to provide, whatever plugins run.
  If a plugin reports such a name as `null`, or fails, the client's own value MUST remain.
  Otherwise a plugin's faulty detection could remove a name a CEP requires to be present.
- **Overrides of standardized names follow the CEP that standardizes them.**
  `CONDA_OVERRIDE_ARCHSPEC` sets a build string, `CONDA_OVERRIDE_UNIX` has no effect, and an empty `CONDA_OVERRIDE_CUDA_ARCH` means absent; those rules are unchanged, and a plugin's verdict is subject to them like the client's own detection would be.

A client MUST NOT attribute a plugin-reported virtual package to its own detection, and vice versa, in anything it records for later (provenance, diagnostics, lockfile metadata if any).

### Overrides

CEP 30 lets `CONDA_OVERRIDE_<NAME>` stand in for a virtual package the client detects.
A client implementing this CEP MUST extend the same mechanism to every declared name that no CEP standardizes, using the [override variable](#virtual-package-names) of the name.
There is nothing further to qualify: the name identifies the verdict on its own, because only one registration answers for it.

- The value MUST be read as a version string, optionally followed by `=` and a build string.
  Without the `=` part the build string is `0`.
- An **empty** value MUST mean the virtual package is **absent**, the sense [CEP 46](./cep-0046.md) gives an empty `CONDA_OVERRIDE_CUDA_ARCH`.
  This is the only way to ask a client to behave as though hardware were missing.
- A value that cannot be read MUST be an error rather than a warning.
  Continuing with a detected value would look to the user like the override took effect.

An overridden name is not wanted, so a registration none of whose declared names is wanted is not live and MUST NOT be run.
Rewriting the answer after paying for it would defeat the purpose: being able to avoid running a plugin at all is what makes overrides useful on a machine that lacks the hardware, or in CI, or when reproducing a bug report.

When a registration is live but some of its declared names are overridden, the override wins for every name it names, and the plugin's verdicts fill in the names left over.
A verdict for an overridden name MUST be discarded rather than merged with the override, so a plugin cannot contribute a build string to a version the user supplied, or the reverse.

A client MUST NOT record an overridden value as though a plugin had produced it.

### Which plugins have to run (the demand scan)

A client SHOULD NOT run a plugin whose declared names cannot affect the solve, because a plugin that does not run cannot fail or take time.
A client MAY determine the set of virtual package names that anything in the solve could reference by looking at the `depends` and `constrains` fields of the records it fetched for the solve, whether from full or sharded repodata, and at the specs the user asked for, and treat a registration none of whose declared names appear in it as not live.

This is an optimization with user-visible effect, so:

- A client MUST NOT skip a registration whose declared names the solve could reference.
- A client MUST NOT rely on skipping for correctness: the set is a bound, not an oracle.

### Target platform

A plugin answers for the host platform.
When the target platform of a solve is not the host platform, a client MUST NOT run plugins for that solve.
Every declared name of every registration is then absent unless an [override](#overrides) supplies it, and the client SHOULD warn once per registration it skipped for this reason, naming the override variable that would fill the name in, so that a user solving for another machine learns what to set.

### Caching

Running a plugin can mean installing an environment and querying hardware.
Clients SHOULD cache verdicts.
This CEP constrains how.

The `cache` object of a report MAY contain:

- `ttl_seconds: int | "REBOOT"`.
  How long the verdicts may be reused.
  The special value `"REBOOT"` means the verdicts may be reused until the next system reboot.
- `watch_paths: list[str]`.
  Absolute paths whose existence or modification time invalidates the verdicts.
  A relative path is malformed.
- `watch_env: list[str]`.
  Environment variable names whose value, in the client's own environment, invalidates the verdicts.

A client that caches:

- MUST key every entry on the registration's identity, its declared names, and the digest of the closure, so that a change in the registration or in any package installed alongside the plugin produces a new entry rather than a stale hit.
- MUST give every entry an expiry.
  "Cache this forever" MUST NOT be expressible: a driver upgrade would otherwise go unnoticed until someone cleared a cache by hand.
- SHOULD apply a default expiry of **1 hour** when a plugin specifies no `ttl_seconds`, a duration long enough to cover the solves of one working session and short enough that a hardware change is noticed the same day.
- MUST NOT honor an integer `ttl_seconds` longer than **30 days**, clamping to that maximum.
- MUST treat `"REBOOT"` as expiring at the next system reboot or after 30 days, whichever comes first.
  How a client observes a reboot boundary is its own business; a boot identifier or boot time are the usual means.
- MUST fall back to a short duration when it cannot observe reboot boundaries.
  This fallback SHOULD be **1 hour**, for the reason the default is.
- MUST treat an entry as expired when any watched path has come into existence, ceased to exist, or changed its modification time since the entry was written, or when any watched variable has a different value.
  A path the client cannot examine counts as absent.
  These conditions expire an entry **sooner** than its TTL, never later.
- MUST treat an integer `ttl_seconds` of `0` as "do not reuse", which is how a plugin whose answer can change at any moment declares itself.
- MUST offer the user a way to discard cached verdicts and cached detector environments without touching anything else.
  A cache a user cannot clear is a fact about the machine the user cannot correct.

Whether a client keeps detector environments around between runs is its own business, subject to [Re-resolution](#re-resolution).

### What a client tells the user

A client MUST record, for every verdict it uses, which registration produced it and the digest of the closure it came from.
It MUST be able to show a user, on request, the registrations it knows, the closure behind each digest, and the verdicts each plugin gave.
A user cannot review a closure a client will not name.

When the digest for a registration differs from the one the client last ran for that registration, the client MUST report that, naming both digests.
A registration source MAY require more, such as a confirmation; see [Security considerations](#security-considerations).

A client MUST let a user disable a registration, identified by origin and plugin name, in persistent configuration, so that a misbehaving plugin can be switched off without editing the registration source.
A disabled registration is not live.

## Security considerations

**This CEP describes a mechanism by which a registration causes third-party code to be executed on the user's machine, before any solve completes and before any package is installed.**
A registration source is therefore a statement about trust, and it MUST say whose consent a registration carries and how a user withholds it.
This CEP only fixes what is run, how it is bounded, and what the client has to be able to show.

### The trusted unit is the environment, not the package

A registration names one package, and it is tempting to read the exposure as that package.
It is not.
A client extends trust to the **closure** of the detector environment, because more than the registered executable gets to run:

- **Dependencies.** A plugin's dependencies are resolved and installed with it, and the detector loads their libraries and calls their helpers.
  A one-line detector that shells out to a vendor tool is trusting that tool, not merely shipping it.
- **Activation.** [Running a plugin](#running-a-plugin) requires the environment's activation to run, which executes the activation scripts of every package that ships one.
  This is deliberate, a detector that needs `LD_LIBRARY_PATH` set has no other way to get it, and it is also code execution that precedes the detector.
- **Byte compilation.** Installing a `noarch: python` package runs the closure's interpreter over the closure's files, unless the client skips it.
- **Updates.** A dependency specified as a range brings in whatever satisfies it later.
  The closure a user saw once is not the closure that runs next month unless something pins it.

Link scripts are deliberately not on that list: [Installation](#installation) forbids running them.

Consequences:

- Anything a registration source lets a user pin MUST be expressed over the digest, because the digest is the identity that corresponds to what runs.
  A coarser approval, of a registrant or of a plugin name, is legitimate, but a client MUST present it as what it is: approval of whatever closure resolves, now and later.
  Approving the name `rocm-detect` while its dependencies are unconstrained approves considerably more than a reviewer of that name would expect.
- A signature requirement, where a registration source or a client imposes one, covers only the artifacts it is checked on; signing the plugin alone leaves its dependencies unattested.
- Scoping resolution to the registration's resolution channels is a supply-chain bound, not only a determinism one: it keeps the closure to code the registrant can vouch for.

PEP 817 reaches the same conclusion for variant providers, treating the provider and its dependencies together as the supply-chain risk; see [Related work](#related-work).

### What is and is not bounded

What limits the exposure:

- The environment is installed from the resolution channels only, and is used for detection only, never for the environment being solved.
- Detector execution and activation are bounded in wall-clock time, and detector output in size.
- A plugin can only contribute its declared names, and the client checks that.
- A plugin's effect on the solve is limited to adding, changing or withholding virtual packages.

What does **not** limit it: everything in the closure is arbitrary code running with the user's privileges.
It can read the user's files, make network requests, and persist.
The bounds above constrain what the detector can *report*, not what it can *do*.

Consequently:

- A registration source MUST define what consent a registration carries, and a client MUST NOT run a plugin for which that consent is not given.
- [What a client tells the user](#what-a-client-tells-the-user) is the minimum visibility every client owes; a client MAY show more.
- A registrant SHOULD treat a plugin as security-relevant code and SHOULD sign it where the ecosystem's signing mechanisms permit (see [CEP 27](./cep-0027.md)), because a plugin is the one package in a channel that runs before the user has seen a transaction.

## Open questions

1. **Lockfile representation.**
   [CEP 37](./cep-0037.md) lockfiles record virtual packages only as dependencies of locked packages, never as the values a solve saw.
   Whether a lockfile should record the verdicts a solve used, and whether the registration and digest belong next to them, is a question for a revision of CEP 37 rather than for this CEP.
   [What a client tells the user](#what-a-client-tells-the-user) requires a client to keep the information, so a lockfile format can pick it up later.

## Future work

- A standard way for a plugin to report *why* it decided what it did, for diagnostics, without turning the report into a log.
- A registration source in client configuration, so that a user can run a plugin no channel registers, or a build of their own choosing.
  Such a source would reuse this CEP unchanged; the shape of the registration and its consent model are what it would have to define.

## Rejected ideas

### A declarative check instead of an executable

A declarative language for "is this hardware present" would avoid arbitrary code execution.
It was considered and set aside because real detection is `ioctl`s, library probes, and vendor tools.
The useful cases are exactly the ones a declarative form would not cover, and a form expressive enough to cover them would be a programming language with extra steps.

This CEP trades safety for capability, and mitigates rather than removes the cost.

### WASM

WASM is sandboxed by default, so that would indeed be a good fit.
The challenge would be to foresee and support all the APIs plugin writers might need so we can make them available to the WASM runtime.
WASM requires a language that gets compiled down to WASM binaries.
We expect many detection plugins to be short shell or Python scripts, which are easier for the "normal user" to examine in an environment.

### Letting a plugin report any virtual package it likes

Dropping the contract check would simplify clients.
Rejected: the declared names are the only part of the arrangement a user can inspect *before* code runs, and a plugin that can report anything makes them worthless.

### Letting a plugin enumerate its own names

A second invocation mode, say `<plugin> --names`, would let a registration consist of a plugin name alone.
Rejected: it makes the contract a check of the plugin against itself, costs every plugin author a second protocol, and costs every run a second process.
A registration source that does not know a plugin's names can ask the party publishing it to write them down.

### Letting a plugin remove a CEP 30 virtual package

Treating a `null` verdict for a standardized name as authoritative.
Rejected: CEP 30 places the obligation on the client, and a plugin's faulty detection would otherwise be able to discharge an obligation that is not the plugin's to discharge.

### An output bound that scales with the number of declared names

An earlier draft scaled the output bound with the size of the registration, on the argument that a plugin answering for ten names has more to say than one answering for one.
Rejected once the numbers were written down: a verdict is a few hundred bytes at most, so ten of them are a few kilobytes, and what actually takes space is diagnostics and cache hints, neither of which grows with the number of names.
A single number is easier for a plugin author to design against, and [Size limits](#size-limits) keep the report itself small independently of it.

### Bounding only standard output

Applying the output bound only to standard output would match the fact that only stdout carries the report.
Rejected because stderr is still data the client has to buffer if it wants to preserve diagnostics for failures.
Counting stdout and stderr together keeps diagnostics useful without letting a plugin exhaust client memory by writing an unbounded diagnostic stream.

### Running link scripts in detector environments

CEP 32 and CEP 34 make link scripts an obligation of the installer, and an earlier draft honored that in detector environments too.
Rejected: it made *installing* a plugin an act of code execution separate from *running* it, which no bound in this CEP covers, and the plugins we expect have no use for it.
A detector that needs setup can do it in its activation script, which is bounded and runs where the user expects code to run.

### Executing plugins during `get_candidates`

Resolving a virtual package at the moment the solver first needs it would be narrower than any demand scan.
Rejected as a requirement: it forces every implementation's solver to be able to await arbitrary work mid-solve, which not all can, and the gain over a demand scan is small.

## Rationale

### Why an executable named after the package

The registration has to name the code to run, and a package name is the one identifier a registration source, a client and a user all share.
Making it the executable's name too means a registration needs no entry-point metadata of its own, and a user who wants to see what runs can look at one file.

### Why absence is `null` rather than omission

A plugin must be able to say "not here", and a client must be able to tell that from "this plugin is broken".
Making absence explicit turns a whole class of plugin bugs (early return, a swallowed error) into a reported contract violation instead of a silently missing capability.

### Why a failure discards every verdict of the plugin

A plugin that violated its contract for one name, or crashed after writing half a report, has told the client that its output cannot be trusted.
Keeping the verdicts that happened to look fine would make the solve depend on which half of a broken report arrived, and would hide the failure behind a solve that mostly works.

### Why a plugin may replace a CEP 30 value

CEP 30 tells a client how to detect `__cuda` because somebody had to write it down once for every client.
A registrant shipping CUDA packages, or the vendor itself, can know more than that text: a newer driver API, a platform the client's authors could not test on, a case where the standard method is wrong.
Letting a plugin's verdict replace the client's value is how that knowledge reaches every client without a CEP per correction, and the standardized name keeps its meaning because the registrant chose to answer for exactly that name.
What a plugin cannot do is remove the name, because the client's obligation to provide it is not the registrant's to discharge.

### Why a mandatory cache expiry

A cache without an expiry turns a one-off detection into a permanent fact about a machine.
`"REBOOT"` is still an expiry: it covers facts that are expected to stay true for a boot session but may change after hardware, driver, kernel, or daemon state is reinitialized, and the 30-day clamp keeps a machine that never reboots from keeping a verdict forever.
Clients that cannot observe reboot boundaries fall back to a short duration rather than treating the value as permanent.
`watch_paths` and `watch_env` make an entry expire sooner, which is the safe direction.

### Why the timeout numbers

Detection blocks a solve, which blocks a user.
A configurable timeout with no ceiling is a configurable hang; the ceiling is what keeps "this plugin is slow" from becoming "the client hung".

The default is set where it is because the measurements we have do not support a smaller one.
conda's own CUDA detector waits up to 60 seconds for a subprocess.
rattler's CUDA probe takes 1.5 seconds on an idle GPU on Windows before any interpreter has started, and a Python interpreter on a loaded Windows CI runner can take longer than 5 seconds to print a line.
A default that fails on slow CI produces failures that look like solve failures, which is worse than waiting.

The ceiling is ten times the default because it is not a default: a user who raises the timeout has a slow machine or a slow plugin and has decided to wait for it.
Five minutes is long enough for any detection we know of on any hardware we know of, and short enough that a hung plugin is still recognizably a hang.

### Why plugins do not run for another target platform

An earlier draft left this open.
Detection is about the machine the plugin runs on, so a plugin built for `linux-64` cannot say anything about a `win-64` solve, and a client that ran the host's plugin and used its verdict for another platform would be inventing an answer.
Treating the names as absent, with overrides as the way to supply them, is the one behavior a user can predict and reproduce.

### Why the digest is specified

A client could keep any identifier it liked for a detector environment, but a user who pins one in configuration, or reports one in a bug, needs it to mean the same thing in every client.
The rendering is chosen so that two clients that resolved the same records get the same digest, whatever order their solvers returned them in.

## Related work

### conda's Python plugin hook

[CEP 4](./cep-0004.md) gave conda a plugin manager, and conda ships a `conda_virtual_packages` hook on it.
A plugin is a Python distribution with an entry point in the `conda` group, discovered in the environment conda runs from, and yields name, version and build string with the same `CONDA_OVERRIDE_*` handling this CEP keeps.
The hook is where the shape of a verdict in this CEP comes from.
What it cannot offer is a plugin that runs in a client that is not conda, or a plugin resolved from a channel like the packages it describes, and those two gaps are what this CEP fills.
An earlier proposal for a language-neutral plugin, an executable on `PATH` printing JSON, was withdrawn in favor of this one.

### PEP 817, wheel variants

The Python packaging ecosystem is designing a mechanism of the same shape.
[PEP 817](https://peps.python.org/pep-0817/), *Wheel Variants: Beyond Platform Tags*, lets a package declare a **variant provider**: metadata advertises a provider package, and an installer may resolve that provider into an isolated environment and run it to detect system capabilities before it selects an artifact.
Its static half has since been split out as [PEP 825](https://peps.python.org/pep-0825/), leaving provider execution and trust in the still-draft PEP 817, which is roughly the split between this CEP and its registration sources.

The two differ in how much the plugin decides.
A PEP 817 provider owns a property namespace: it defines the vocabulary, it says which properties are compatible with the running system, and it supplies the priority order used to rank variants against one another.
The provider is part of the selection algorithm.

A plugin in this CEP defines none of that.
It reports virtual packages, which have a name, a version and an optional build string like any other package, and everything downstream is CEP 29 MatchSpec and the ordinary solver.
The narrow scope is what keeps the mechanism reviewable: a plugin's entire influence on a solve is a set of `name`/`version`/`build_string` triples, which is also why the [contract](#the-contract) can be checked mechanically.

Both proposals make resolution depend on running third-party code, and both treat that as a security-relevant step.
PEP 817 requires that installers "MUST NOT install or run provider packages, unless they can determine the particular provider package version to be trusted", leaves the mechanism to implementations, and expects a vetted pool of providers so that common ones need no explicit opt-in.
PEP 817 also draws the supply-chain boundary around the provider *and its dependencies*, which is the conclusion this CEP reaches in [The trusted unit is the environment, not the package](#the-trusted-unit-is-the-environment-not-the-package).

## References

- [CEP 4 - Implement initial conda plugin mechanism](./cep-0004.md)
- [CEP 20 - Support for `abi3` Python packages](./cep-0020.md)
- [CEP 26 - Identifying Packages and Channels in the conda Ecosystem](./cep-0026.md)
- [CEP 27 - Standardizing a publish attestation for the conda ecosystem](./cep-0027.md)
- [CEP 29 - The `MatchSpec` query language](./cep-0029.md)
- [CEP 30 - Virtual packages](./cep-0030.md)
- [CEP 32 - Management and structure of conda environments](./cep-0032.md)
- [CEP 33 - Version literals and their ordering](./cep-0033.md)
- [CEP 34 - Contents of conda packages](./cep-0034.md)
- [CEP 37 - `conda-lock.yml` lockfiles](./cep-0037.md)
- [CEP 46 - The `__cuda_arch` virtual package](./cep-0046.md)
- [PEP 817 - Wheel Variants: Beyond Platform Tags](https://peps.python.org/pep-0817/)
- [PEP 825 - Wheel Variants: Package Format](https://peps.python.org/pep-0825/)

## Copyright

All CEPs are explicitly [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).

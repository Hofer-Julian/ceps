# CEP XXXX - Virtual package detection plugins

<table>
<tr><td> Title </td><td> Virtual package detection plugins </td></tr>
<tr><td> Status </td><td> Draft </td></tr>
<tr><td> Author(s) </td><td> Julian Hofer &lt;julian@prefix.dev&gt;, Wolf Vollprecht &lt;wolf@prefix.dev&gt;, Tobias Hunger &lt;tobias@prefix.dev&gt;</td></tr>
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
A client can therefore only detect the names it was written to support; packages cannot depend on detection of other system capabilities.

This CEP defines a *detection plugin*: an ordinary conda package with an executable of the same name that reports virtual packages as JSON on standard output.
It specifies installation in an isolated environment, execution and resource bounds, report validation, and how verdicts enter the solve, are cached, and are overridden.

A plugin can be written in any language, and any client that can install a conda package and start a process can run one.
A *registration source* determines which plugins to run.
[CEP XXXX (Channel-provided virtual package plugins)](./cep-XXXX.md) defines the first registration source: a channel registering plugins in its `repodata.json`.

## Motivation

The set of virtual packages in CEP 30 is closed, and extending it requires a CEP per name (as [CEP 46](./cep-0046.md) did for `__cuda_arch`).
That process suits capabilities shared across the ecosystem, but is less practical for:

- **Accelerator and interconnect stacks.** ROCm, oneAPI, Metal, Infiniband, TPUs and NPUs each need version and capability detection.
  These matter to the channels shipping builds against them.
- **Site-specific capability.** An organization's internal channel may need to detect a license server, a filesystem, a kernel module or a CPU feature that public channels do not use.
- **Capabilities that change faster than clients release.** A new GPU generation is a new value for an existing name.
  Today every client has to release an update to detect it, even when the party building for it already knows how.

conda provides a virtual package hook through the plugin manager introduced by [CEP 4](./cep-0004.md).
It is a Python interface and cannot be used by mamba or Pixi.
Conda discovers plugins in the Python environment it runs from.
A plugin can be installed as an ordinary dependency, but adding it to the environment being solved does not make it available to the conda process performing that solve.
The conda-forge external MPI detector uses this hook and works only in conda.

A client-independent plugin definition and execution protocol would let any registration source register a detector and let every client reach the same verdict on the same machine.

## Specification

### Terminology

- A **plugin** is a conda package that satisfies [The plugin package](#the-plugin-package).
- A **registration** tells a client to run one plugin.
  It consists of:
  - the **origin**: an identifier for where the registration came from, whose form the registration source defines (for a channel, its base URL);
  - the **plugin name**: the name of the plugin package, which is also the name of the executable to run;
  - the **declared names**: a non-empty set of valid virtual package names the plugin answers for;
  - the **resolution channels**: an ordered, non-empty list of channels the plugin and its dependencies are resolved from.

  The pair of origin and plugin name identifies a registration for purposes such as disabling or pinning it.
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
  They are usually, but not necessarily, the same.
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
The registration source defines how to handle such a name.

### The plugin package

A plugin is an ordinary conda package.

- It MUST contain an executable whose file name is the plugin name in its normalized (lowercase) form, in one of the directories that [CEP 32](./cep-0032.md) puts on `PATH` when the environment is activated.
  On Windows the file MUST carry an `.exe`, `.cmd` or `.bat` extension.
- It MAY have dependencies.
- It MUST NOT depend, directly or through its dependencies, on virtual packages other than those CEP 30 and its extensions oblige the client to provide.
  Detection of other virtual packages depends on running plugins, so those packages cannot be prerequisites.
- It and its dependencies MUST NOT rely on pre-link, post-link or pre-unlink scripts (CEP 34), because a client does not run them in a detector environment (see [Installation](#installation)).
- It SHOULD have as few dependencies as possible, and SHOULD pin them, because each dependency adds code the user runs and can change the digest; see [Security considerations](#security-considerations).

The plugin MUST be resolvable from the registration's resolution channels for the host platform.
A plugin that cannot be resolved is a failed plugin (see [Failure](#failure)), not a broken registration: the client can only discover this by attempting resolution.

### The detector environment

#### Resolution

A client MUST resolve the plugin by a [CEP 29](./cep-0029.md) MatchSpec consisting of the plugin name and nothing else, except that a registration source MAY require the MatchSpec to carry a channel qualifier naming one of the resolution channels.
It MUST resolve against the registration's resolution channels in the given order, for the host platform, loading the `noarch` subdir alongside the host subdir.
The solver's ordinary preference for the newest version therefore decides which build of the plugin runs.
A registration source MUST NOT add version or build constraints to that MatchSpec; a registrant that wants a different build has to serve it.

A client MUST NOT resolve the plugin or its dependencies from any channel that is not among the resolution channels.
The registration source chooses those channels, allowing the registrant to restrict the code the plugin can pull in.

The virtual packages available to that resolution are the client's own CEP 30 virtual packages, honoring `CONDA_OVERRIDE_*`, and nothing a plugin reported.

Resolution is expected to be cheap because it uses repodata already loaded for the solve and downloads nothing.
Caching avoids installation and execution, not resolution; see [Re-resolution](#re-resolution).

#### Installation

A client MUST install the closure into an environment that is used for nothing else, following the environment layout of [CEP 32](./cep-0032.md) and the package layout of [CEP 34](./cep-0034.md).

- It MUST NOT install a plugin into the environment being solved for, and MUST NOT make a plugin a dependency of it.
  The plugin informs the solve; requiring the solve to include its detection tooling would be circular.
- It MUST NOT install a plugin into an environment the client itself runs from, or into any environment a user works in.
  Unrelated changes in those environments would invalidate the digest as an identity for the plugin and its dependencies.
- Notwithstanding CEP 32 and CEP 34, it MUST NOT execute pre-link, post-link or pre-unlink scripts of any package in the closure.
- It MAY skip the byte compilation of `noarch: python` packages ([CEP 20](./cep-0020.md)), which runs the closure's own Python interpreter over the closure's own files at install time.
  A client that performs it SHOULD bound it as it bounds activation.
  Trusting a registration therefore allows execution of the executable, the libraries it loads, the closure's activation scripts, and at most that byte compilation.
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
Two artifacts of one build in different formats (`.tar.bz2` and `.conda`) have different hashes and therefore different digests because they are different files.

The digest identifies the closure for caching, reporting, pinning and approval; see [Security considerations](#security-considerations).

#### Re-resolution

A client MUST NOT run a plugin from a detector environment it looked up by registration alone.
Before running a plugin, and before consulting its verdict cache for that registration, a client MUST resolve the registration against current repodata and compute the digest.
It MAY then serve a cached verdict whose key matches (see [Caching](#caching)), and otherwise MUST reuse an existing environment only if the digest matches, installing a new one if not.

When a registrant publishes an update that resolves to a new closure, its digest changes and the old closure's verdicts are not reused.

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
A plugin that exits `0` with only whitespace on standard output has also failed.
This includes plugins that mistakenly write their report to standard error; the client does not interpret this as "nothing detected".

#### Bounds

A client MUST bound a plugin run.

- **Time.** A client MUST apply a timeout to the detector process and MUST terminate a process that exceeds it.
  It SHOULD also terminate the process's descendants, since a helper can keep the output pipe open after the detector times out.
  The clock starts when the process is spawned; it excludes resolution, installation and activation.
  The timeout SHOULD default to **30 seconds**.
  A client MAY let a user raise it, but MUST NOT allow any timeout longer than **300 seconds**, to limit how long detection can delay the start of a solve.
  The activation step MUST be bounded separately, with the same default and ceiling, and a timed-out activation is a plugin failure.
- **Output.** A client MUST stop reading once **1 MiB** (1,048,576 bytes) of standard output and standard error combined has been read, MUST terminate the process if it is still running, and MUST treat the plugin as failed.
  The bound is the same for every plugin.
  A valid report is small (see [Size limits](#size-limits)), so this bound primarily limits diagnostics.

Diagnostics a plugin wrote to standard error before a bound was exceeded MUST be kept and reported when the run fails.
This preserves information for diagnosing and reporting failures without allowing an unbounded diagnostic stream to exhaust client memory.

#### Failure

A plugin has failed when the plugin cannot be resolved or installed, activation fails or times out, the executable is missing, the run exceeds a bound, the process exits non-zero, the report is empty or malformed, the plugin violates [the contract](#the-contract), or the registration source declares the registration failed (for example a pinned digest that does not match).

- A failed plugin MUST NOT abort the solve on its own.
- A failed plugin's verdicts MUST NOT enter the solve, including verdicts for names it did report correctly.
  Every declared name of the registration is absent for that solve, unless an [override](#overrides) supplies it or CEP 30 obliges the client to provide it, in which case the client's own value stands (see [Interaction with client-detected virtual packages](#interaction-with-client-detected-virtual-packages)).
  The solver then reports an unsatisfiable dependency on an absent name in the ordinary way.
- A client MUST report every plugin failure to the user, together with the diagnostics it captured, whether or not the solve succeeds.
  A failure may not affect the current solve but may explain a later unsatisfiable dependency.

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
  A missing or unsupported version is a plugin failure because the client cannot interpret the remaining keys.
- `virtual_packages: dict[str, dict | null]`.
  Required.
  One entry per virtual package the plugin gives a verdict about, keyed by name; keys are compared in normalized form.
  - A **dictionary** value means the virtual package is present.
    It MUST contain `version: str`, a version string as defined by CEP 33, and MAY contain `build_string: str`, which MUST satisfy the build string rules of CEP 26.
    An absent `build_string` means `0`, the default CEP 30 gives virtual packages.
    The CEP standardizing a name decides which information belongs in the version and which in the build string.
    For a name no CEP standardizes, the registrant decides, and SHOULD put a version in the version so that ordinary version constraints work on it.
  - A **`null`** value means the virtual package is not present on this system.
    This explicit verdict differs from an omitted name.
- `cache: dict`.
  Optional.
  See [Caching](#caching).

Keys other than these MUST be ignored, at both the top level and inside a `virtual_packages` entry.
This allows clients to use the parts of a later protocol revision they understand.
Rejecting unknown keys would not prevent code execution: the client has already run the plugin.
A known key with the wrong value type, including a negative or non-integer `ttl_seconds`, makes the report malformed.

Because the report is keyed by name, a duplicate verdict cannot be expressed, and clients therefore need no rule for one.

#### Size limits

A report MUST satisfy the following, measured on the decoded string values, and a report that does not is malformed:

- `version` and `build_string` of a verdict: at most 256 bytes each.
- `watch_paths` and `watch_env`: at most 32 entries each, every entry at most 4096 bytes.

The [output bound](#bounds) applies regardless: a report that satisfies these limits but, through escaping or unknown keys, exceeds the bound has still failed.

### The contract

A client MUST check that the report matches the registration before passing any verdict to the solver:

- A plugin MUST give a verdict for every declared name of its registration.
  A declared name absent from `virtual_packages` is a contract violation; `null` indicates an absent capability.
- A plugin MUST NOT report a name that is not among its declared names.
  Doing so is a contract violation, and the client MUST NOT let the undeclared name reach the solve.

A contract violation is a plugin failure, with the consequences of [Failure](#failure): all of the plugin's verdicts are discarded, and the client reports it.
Declared names let the user inspect a plugin's possible contributions to the solve before running it.

A client MUST hold a plugin to the full contract even when only some of its declared names are wanted, and MUST discard the verdicts for names that are not wanted afterwards.
The plugin cannot know which names the client assigned to other registrations or the user overrode.

### Verdicts in the solve

A registration source guarantees at most one registration per name in a solve.
Each verdict enters the solve as a client-detected virtual package would:

- A **dictionary** verdict contributes one virtual package record, with that name, version and build string, to the candidate pool.
- A **`null`** verdict contributes no record, and the name is absent for the whole solve.

A MatchSpec on a plugin-provided name matches that one record as specified by CEP 29.
This requires no changes to MatchSpec, solvers, or the handling of specs from a command line or manifest, because each name has only one candidate.

Records from all channels use the same verdict, regardless of which channel registered the plugin.
A registrant that wants only some packages to use a name has to choose a distinctive name; verdicts are not scoped.

### Interaction with client-detected virtual packages

CEP 30 requires the client to provide standardized virtual packages and specifies several of their values.
This CEP allows plugins to replace detected values while retaining the other requirements:

- **A plugin verdict for a standardized name MAY replace the client's own value.**
  Where CEP 30 or a later CEP requires a client to set a standardized name to a detected value, a client implementing this CEP MUST use a plugin's dictionary verdict for that name instead, when a live registration produced one.
  This lets a registrant provide an improved detector for a shared capability.
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
The name alone identifies the verdict because only one registration answers for it.

- The value MUST be read as a version string, optionally followed by `=` and a build string.
  Without the `=` part the build string is `0`.
- An **empty** value MUST mean the virtual package is **absent**, the sense [CEP 46](./cep-0046.md) gives an empty `CONDA_OVERRIDE_CUDA_ARCH`.
  This is the only way to ask a client to behave as though hardware were missing.
- A value that cannot be read MUST be an error rather than a warning.
  Continuing with a detected value would conceal that the override was not applied.

An overridden name is not wanted, so a registration none of whose declared names is wanted is not live and MUST NOT be run.
This lets users avoid running plugins on machines without the hardware, in CI, or when reproducing a bug report.

When a registration is live but some declared names are overridden, the overrides take precedence and the plugin supplies verdicts for the remaining names.
A verdict for an overridden name MUST be discarded rather than merged with the override, so a plugin cannot contribute a build string to a version the user supplied, or the reverse.

A client MUST NOT record an overridden value as though a plugin had produced it.

### Which plugins have to run (the demand scan)

A client SHOULD NOT run a plugin whose declared names cannot affect the solve, to avoid unnecessary delays and failures.
A client MAY determine the set of virtual package names that anything in the solve could reference by looking at the `depends` and `constrains` fields of the records it fetched for the solve, whether from full or sharded repodata, and at the specs the user asked for, and treat a registration none of whose declared names appear in it as not live.

Because this optimization affects which plugins run:

- A client MUST NOT skip a registration whose declared names the solve could reference.
- A client MUST NOT rely on skipping for correctness: the scan provides a bound on the names a solve could reference, not an exact prediction.

### Target platform

A plugin answers for the host platform.
When the target platform of a solve is not the host platform, a client MUST NOT run plugins for that solve.
Every declared name of every registration is then absent unless an [override](#overrides) supplies it, and the client SHOULD warn once per registration skipped for this reason, naming the override variable to set for a solve on another machine.

### Caching

Running a plugin can mean installing an environment and querying hardware.
Clients SHOULD cache verdicts.
The following rules govern reuse and invalidation.

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

- MUST key every entry on the registration's identity, its declared names, and the digest of the closure.
  A change to the registration or any package installed with the plugin then produces a new entry rather than a stale hit.
- MUST give every entry an expiry.
  Indefinite caching MUST NOT be expressible, because a driver upgrade could otherwise go unnoticed until the cache was cleared manually.
- SHOULD apply a default expiry of **1 hour** when a plugin specifies no `ttl_seconds`, to cover solves in one working session while detecting hardware changes within the same day.
- MUST NOT honor an integer `ttl_seconds` longer than **30 days**, clamping to that maximum.
- MUST treat `"REBOOT"` as expiring at the next system reboot or after 30 days, whichever comes first.
  The client chooses how to observe reboot boundaries, usually with a boot identifier or boot time.
- MUST fall back to a short duration when it cannot observe reboot boundaries.
  This fallback SHOULD be **1 hour**, for the same reason as the default.
- MUST treat an entry as expired when any watched path has come into existence, ceased to exist, or changed its modification time since the entry was written, or when any watched variable has a different value.
  A path the client cannot examine counts as absent.
  These conditions expire an entry sooner than its TTL, never later.
- MUST treat an integer `ttl_seconds` of `0` as "do not reuse", for plugins whose verdicts can change at any time.
- MUST offer the user a way to discard cached verdicts and cached detector environments without touching anything else.
  This lets the user correct stale detection without clearing unrelated data.

Clients decide whether to retain detector environments between runs, subject to [Re-resolution](#re-resolution).

### What a client tells the user

A client MUST record, for every verdict it uses, which registration produced it and the digest of the closure it came from.
It MUST be able to show a user, on request, the registrations it knows, the closure behind each digest, and the verdicts each plugin gave.

When the digest for a registration differs from the one the client last ran for that registration, the client MUST report that, naming both digests.
A registration source MAY require more, such as a confirmation; see [Security considerations](#security-considerations).

A client MUST let a user disable a registration, identified by origin and plugin name, in persistent configuration, so that a misbehaving plugin can be switched off without editing the registration source.
A disabled registration is not live.

## Security considerations

A registration causes third-party code to run on the user's machine before a solve completes and before any package is installed.
A registration source MUST say whose consent a registration carries and how a user withholds it.
This CEP specifies what runs, its resource bounds, and what information the client provides.

### The trusted unit is the environment, not the package

Trust in a registration extends to the whole closure of the detector environment, not just the named executable:

- **Dependencies.** A plugin's dependencies are resolved and installed with it, and the detector loads their libraries and calls their helpers.
  A detector that calls a vendor tool also trusts that tool.
- **Activation.** [Running a plugin](#running-a-plugin) executes the activation scripts of every package that ships one.
  This supports detectors that need settings such as `LD_LIBRARY_PATH`, but also runs code before the detector itself.
- **Byte compilation.** Installing a `noarch: python` package runs the closure's interpreter over the closure's files, unless the client skips it.
- **Updates.** A dependency specified as a range can resolve to a different version later.
  Without pinning, a closure the user reviewed may differ from one run later.

[Installation](#installation) forbids running link scripts.

Consequences:

- Anything a registration source lets a user pin MUST be expressed over the digest, because the digest is the identity that corresponds to what runs.
  A client MUST present coarser approval of a registrant or plugin name as approval of whatever closure resolves, now and later.
  For example, approving `rocm-detect` with unconstrained dependencies approves future versions of those dependencies too.
- A signature requirement, where a registration source or a client imposes one, covers only the artifacts it is checked on; signing the plugin alone leaves its dependencies unattested.
- Restricting resolution to the registration's resolution channels limits the supply chain to code the registrant can vouch for, as well as making resolution deterministic.

PEP 817 also treats the provider and its dependencies together as the supply-chain risk; see [Related work](#related-work).

### What is and is not bounded

The mechanism limits installation, execution time, output size and contributions to the solve:

- The environment is installed from the resolution channels only, and is used for detection only, never for the environment being solved.
- Detector execution and activation are bounded in wall-clock time, and detector output in size.
- A plugin can only contribute its declared names, and the client checks that.
- A plugin's effect on the solve is limited to adding, changing or withholding virtual packages.

These restrictions do not sandbox the closure, which runs arbitrary code with the user's privileges.
It can read the user's files, make network requests, and persist.

Consequently:

- A registration source MUST define what consent a registration carries, and a client MUST NOT run a plugin for which that consent is not given.
- [What a client tells the user](#what-a-client-tells-the-user) defines the minimum information clients provide; a client MAY show more.
- A registrant SHOULD treat a plugin as security-relevant code and SHOULD sign it where the ecosystem's signing mechanisms permit (see [CEP 27](./cep-0027.md)), because it runs before the user has seen a transaction.

## Open questions

1. **Lockfile representation.**
   [CEP 37](./cep-0037.md) lockfiles record virtual packages only as dependencies of locked packages, never as the values a solve saw.
   Whether a lockfile should record the verdicts a solve used, and whether the registration and digest belong next to them, is a question for a revision of CEP 37 rather than for this CEP.
   [What a client tells the user](#what-a-client-tells-the-user) requires a client to keep the information, so a lockfile format can pick it up later.

## Future work

- A standard way for a plugin to report *why* it decided what it did, for diagnostics, without turning the report into a log.
- A registration source in client configuration, so that a user can run a plugin no channel registers, or a build of their own choosing.
  Such a source would define the registration's form and consent model, reusing this CEP unchanged.

## Rejected ideas

### A declarative check instead of an executable

A declarative hardware check could avoid arbitrary code execution, but detection often needs `ioctl`s, library probes, and vendor tools.
A declarative form would either exclude these cases or need the expressiveness of a programming language.
This CEP allows executable detection with mitigations for its risks rather than eliminating code execution.

### WASM

WASM is sandboxed by default, but clients would need to anticipate and expose every API a plugin might need.
It also requires a language that compiles to WASM.
Many detection plugins are expected to be short shell or Python scripts, which are easier for users to examine in an environment.

### Letting a plugin report any virtual package it likes

Removing the contract check would simplify clients, but declared names would no longer describe a plugin's possible contributions before code runs.

### Letting a plugin enumerate its own names

A second invocation mode, such as `<plugin> --names`, would let a registration consist of a plugin name alone.
It would check the plugin against its own declarations, require a second protocol from plugin authors, and add a second process to every run.
A registration source can instead ask the publisher to provide the names.

### Letting a plugin remove a CEP 30 virtual package

Treating a `null` verdict for a standardized name as authoritative would let faulty plugin detection remove a name that CEP 30 requires the client to provide.

### An output bound that scales with the number of declared names

An earlier draft scaled the output bound with the number of declared names.
However, a verdict is at most a few hundred bytes, so ten verdicts occupy only a few kilobytes.
Diagnostics and cache hints account for most of the output and do not grow with the number of names.
A fixed bound is easier for plugin authors to work with, while [Size limits](#size-limits) independently limit the report itself.

### Bounding only standard output

Only stdout carries the report, but the client also buffers stderr to preserve diagnostics on failure.
Bounding stdout alone would let a plugin exhaust client memory through stderr.
The combined bound preserves diagnostics while limiting memory use.

### Running link scripts in detector environments

CEP 32 and CEP 34 require installers to run link scripts, and an earlier draft retained that requirement for detector environments.
This allowed code execution during installation without any of the bounds in this CEP, and the expected plugins have no use for it.
Detectors that need setup can use activation scripts, which run during the bounded activation step.

### Executing plugins during `get_candidates`

Resolving a virtual package when the solver first needs it would be narrower than a demand scan.
Requiring this would force solvers to await arbitrary work mid-solve, which not all support, for little gain over a demand scan.

## Rationale

### Why an executable named after the package

The package name is an identifier shared by the registration source, client and user.
Using it as the executable name avoids separate entry-point metadata and lets a user locate the executable in one file.

### Why absence is `null` rather than omission

Explicit absence lets clients distinguish a missing capability from a broken plugin.
An omitted verdict caused by an early return or swallowed error becomes a reported contract violation rather than a silently missing capability.

### Why a failure discards every verdict of the plugin

Output from a plugin that violates its contract or crashes after writing part of a report is unreliable.
Keeping apparently valid verdicts would make the solve depend on which part arrived and could conceal the failure.

### Why a plugin may replace a CEP 30 value

CEP 30 specifies a shared detection method for `__cuda`.
A registrant shipping CUDA packages, or the vendor, may have a newer driver API, access to a platform the client's authors could not test, or knowledge of a case where the standard method is wrong.
A plugin can supply that detection to every client without a CEP per correction, while retaining the standardized name's meaning.
It cannot remove a name the client is required to provide.

### Why a mandatory cache expiry

Without expiry, cached verdicts could remain in use after the hardware or software they describe changes.
`"REBOOT"` covers state expected to remain stable during a boot session but potentially change when hardware, drivers, the kernel, or daemons are reinitialized.
The 30-day clamp prevents indefinite reuse on machines that never reboot.
Clients that cannot observe reboot boundaries fall back to a short duration.
`watch_paths` and `watch_env` can only shorten an entry's lifetime.

### Why the timeout numbers

Detection delays the solve, so a timeout ceiling limits how long a slow or hung plugin can make the user wait.

The available measurements do not support a smaller default.
conda's own CUDA detector waits up to 60 seconds for a subprocess.
rattler's CUDA probe takes 1.5 seconds on an idle GPU on Windows before any interpreter has started, and a Python interpreter on a loaded Windows CI runner can take longer than 5 seconds to print a line.
A shorter default could cause detection failures on slow CI that appear as solve failures.

The ceiling is ten times the default to allow users to wait longer for a slow machine or plugin.
Five minutes covers all detection times and hardware known to the authors while limiting the wait for a hung plugin.

### Why plugins do not run for another target platform

An earlier draft left cross-platform behavior open.
A plugin detects the machine it runs on: a `linux-64` plugin cannot determine the capabilities of a `win-64` target.
Treating declared names as absent and allowing overrides gives users a predictable, reproducible way to supply target values.

### Why the digest is specified

A digest used in configuration pins or bug reports needs to identify the same closure across clients.
The specified rendering produces the same digest for the same records, regardless of solver output order.

## Related work

### conda's Python plugin hook

[CEP 4](./cep-0004.md) introduced conda's plugin manager, which provides a `conda_virtual_packages` hook.
Plugins are Python distributions with an entry point in the `conda` group, discovered in the environment running conda.
They yield a name, version and build string, with the same `CONDA_OVERRIDE_*` handling retained here.
This CEP uses that verdict format but supports other clients and resolves detectors separately from the environment running the client.
An earlier proposal for a language-neutral plugin, an executable on `PATH` printing JSON, was withdrawn in favor of this one.

### PEP 817, wheel variants

[PEP 817](https://peps.python.org/pep-0817/), *Wheel Variants: Beyond Platform Tags*, proposes a similar mechanism for Python packaging.
Package metadata advertises a **variant provider** package, which an installer may resolve into an isolated environment and run to detect system capabilities before selecting an artifact.
The static portion is now [PEP 825](https://peps.python.org/pep-0825/); provider execution and trust remain in the draft PEP 817.
This roughly parallels the separation between this CEP and its registration sources.

The proposals differ in the plugin's role in selection.
A PEP 817 provider owns a property namespace, defines its vocabulary, identifies properties compatible with the running system, and supplies the priority order for ranking variants.
It is part of the selection algorithm.

A plugin in this CEP reports only virtual packages with a name, version and optional build string.
CEP 29 MatchSpec and the ordinary solver handle selection.
This limits the plugin's influence on the solve to `name`/`version`/`build_string` triples and allows mechanical checking of the [contract](#the-contract).

Both proposals make resolution depend on running third-party code, and both treat that as a security-relevant step.
PEP 817 requires that installers "MUST NOT install or run provider packages, unless they can determine the particular provider package version to be trusted", leaves the mechanism to implementations, and expects a vetted pool of providers so that common ones need no explicit opt-in.
PEP 817 also treats the provider and its dependencies as a supply-chain unit, as described here in [The trusted unit is the environment, not the package](#the-trusted-unit-is-the-environment-not-the-package).

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

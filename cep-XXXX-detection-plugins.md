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

A detection plugin is an ordinary conda package with a same-named executable that reports virtual packages as JSON.
This CEP defines its isolated installation, execution, report, solver integration, overrides and caching, independently of the client or implementation language.
Registration sources decide which plugins participate; [CEP XXXX](./cep-XXXX.md) defines channel-provided registrations.

## Motivation

[CEP 30](./cep-0030.md) standardizes virtual packages that clients must detect, but adding a name requires another CEP and new detection requires client updates.
External MPI installations, site-specific license servers or kernel modules, and vendor accelerator stacks need detection that their maintainers can distribute without waiting for every client to release.

conda's [CEP 4](./cep-0004.md) Python plugin hook supports this, but mamba and Pixi cannot use it.
conda discovers hooks in the Python environment running conda, not the environment being solved: installing a hook as a dependency there does not make it available to the current solve.
The conda-forge external MPI detector uses that hook, not the executable protocol proposed here.
A language-independent protocol lets sites, vendors and channels supply detection to all clients without installing tooling into the client itself.

## Specification

### Registrations

A **registration** consists of four fields:

- **Origin:** an identifier defined by the registration source, such as a channel's base URL.
- **Plugin name:** the package name, also used as the executable name, normalized to lowercase.
- **Declared names:** a non-empty set of virtual package names the plugin reports on.
- **Resolution channels:** an ordered, non-empty list of channels from which to resolve the plugin and its dependencies.

Origin and plugin name together identify a registration, including for disabling or pinning it.
A **registration source** is a specification that supplies these fields; this CEP defines none.
It MUST define whose consent a registration carries and how a user withholds it.
It MUST assign each name and override variable to at most one registration per solve, define how collisions are resolved, and define how invalid declared names are handled without passing them to the client.
These assignments determine which results may enter the solve, not which names the plugin must report.

Declared names MUST satisfy [CEP 26](./cep-0026.md), begin with two underscores, contain at most 64 characters, and match:

```re
^__[a-z0-9][._-]?([a-z0-9]+(\.|-|_|$))*$
```

A name's **override variable** is `CONDA_OVERRIDE_` followed by the name without its two leading underscores, uppercased, with `-` and `.` replaced by `_`.
For example, `__acme-rocm` and `__acme.rocm` both map to `CONDA_OVERRIDE_ACME_ROCM`, so a source must resolve that collision even though the names differ.

### The plugin package

A plugin MUST contain an executable named after its normalized package name in one of the [CEP 32](./cep-0032.md) environment `PATH` directories.
On Windows it MUST have an `.exe`, `.cmd` or `.bat` extension.

A plugin MAY have dependencies, but MUST NOT depend directly or transitively on virtual packages other than those CEP 30 and its extensions require clients to provide.
The plugin and its dependencies MUST NOT rely on pre-link, post-link or pre-unlink scripts, which are not run in detector environments.
Plugins SHOULD minimize and pin dependencies: each adds executable code and can change the environment digest.

The plugin MUST be resolvable from its resolution channels for the **host platform**, the machine running the client (CEP 30's native platform).
Failure to resolve is a plugin failure, not a malformed registration.

### Resolution and installation

A client MUST resolve a registration using a [CEP 29](./cep-0029.md) MatchSpec containing only the plugin name, except that a registration source MAY require a channel qualifier naming one of the resolution channels.
A source MUST NOT add version or build constraints; a publisher selecting a different build must serve it.
Ordinary solver preferences, including preference for newer versions, determine the resolved build.

Resolution MUST use the resolution channels in their given order, loading their host and `noarch` subdirs, and MUST NOT use any other channel for the plugin or its dependencies.
The only virtual packages available to this resolution MUST be the client's own CEP 30 virtual packages, honoring `CONDA_OVERRIDE_*`, never plugin results.

Before executing a plugin or consulting its result cache, a client MUST resolve it against current repodata and compute the [environment digest](#environment-digest).
A registration-only environment lookup is insufficient: a client MUST reuse an environment only when its digest matches, otherwise install the newly resolved packages.
Caching avoids installation and execution, not resolution; changed plugin or dependency records prevent reuse of old results.

The **detector environment** is a dedicated conda environment containing the plugin and its resolved dependencies.
Its installation MUST follow CEP 32 and [CEP 34](./cep-0034.md), with these restrictions:

- It MUST be used only for detection, never as the target environment, an environment running the client, or a user's working environment.
  The plugin MUST NOT become a dependency of the target environment.
- The client MUST NOT execute any resolved package's pre-link, post-link or pre-unlink scripts, notwithstanding CEP 32 and CEP 34.
- The client MAY skip byte compilation of `noarch: python` packages ([CEP 20](./cep-0020.md)); if performed, it SHOULD be bounded like activation.
  This executes the resolved Python interpreter over the installed files.
- The client MUST NOT run a plugin from an incomplete installation, and MUST prevent concurrent installations of the same digest from corrupting each other.

Clients MAY share an environment between registrations with identical resolved packages and decide its location and retention, subject to the digest check above.

### Environment digest

A client MUST compute the digest from every resolved package record, including the plugin, as follows:

1. Form `<name>\t<version>\t<build>\t<artifact>` for each record.
   Use the normalized lowercase name and verbatim `version` and `build` strings.
   For `<artifact>`, use non-null `sha256` in lowercase hexadecimal, otherwise non-null `md5` in lowercase hexadecimal, otherwise the file name (`fn`).
2. Sort the lines bytewise ascending.
3. Join with `\n` without a trailing newline, encode as UTF-8, and compute SHA-256, rendered as lowercase hexadecimal.

CEP 26 excludes tabs and newlines from these fields.
Different archive formats of a build have different artifact hashes and therefore different digests.
This is a fingerprint of the resolved records for caching, reporting and exact digest pins, not proof of artifact authenticity or unchanged installed code.
The same records produce the same digest across clients regardless of solver output order.

### Running a plugin

The **target platform** is the platform being solved for.
A client MUST NOT run plugins when it differs from the host platform.
For such a solve, plugin-provided names are absent unless overridden, except for values the client must provide under CEP 30 and its extensions for that target.
The client SHOULD warn once per skipped registration and name its override variables.

A client MUST NOT run a registration disabled by the user, lacking the source's required consent, or with no declared names still assigned to it and not overridden.
Otherwise it obtains results from a valid cache entry or executes the plugin, subject to the following optimization.

A client SHOULD avoid execution when none of a plugin's applicable declared names can affect the solve.
It MAY conservatively scan the `depends` and `constrains` fields of records fetched for the solve, from full or sharded repodata, together with user-requested specs.
It MUST NOT skip a registration whose applicable names the solve could reference, and MUST NOT rely on exact prediction or on skipping for correctness.

To execute a plugin, the client MUST:

1. Evaluate the detector environment's activation scripts as for normal activation, including those of its dependencies, and exclude activation output from the report.
   Failed activation is a plugin failure.
2. Prepend the environment's `PATH` directories in CEP 32 order to the inherited `PATH`.
3. Find the normalized plugin executable only in those directories, in that order.
   On Windows, try `.exe`, `.cmd` and `.bat` in that order within each directory.
   A missing executable is a plugin failure.
4. Start it with no arguments, no input on standard input, and the activated environment; the working directory is unspecified.
5. Capture standard output as the report, separately from standard error as diagnostics.

A successful plugin MUST exit with status `0` and produce a non-whitespace report on standard output.
A nonzero exit, empty output or a report written only to standard error is a plugin failure.

Execution MUST have the following bounds:

- **Time:** the client MUST time out and terminate the process, and SHOULD terminate its descendants.
  The clock starts at process spawn, excluding resolution, installation and activation.
  The timeout SHOULD default to **30 seconds**; clients MAY let users raise it, but MUST NOT allow more than **300 seconds**.
  Activation MUST be bounded separately with the same default and ceiling; its timeout is a plugin failure.
- **Output:** after reading **1 MiB (1,048,576 bytes)** of standard output and standard error combined, the client MUST stop reading, terminate a still-running process, and fail the plugin.
  This bound is fixed for all plugins, including reports containing unknown keys or escaped strings.

The client MUST retain diagnostics captured on standard error before a bound was reached for [failure reporting](#failure-handling).

### The report

A plugin MUST write exactly one UTF-8 JSON object to standard output, with only whitespace around it.
Arrays, additional values and trailing text are malformed.
For a registration declaring `__cuda`, `__cuda_arch` and `__cuda_mps`:

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

The object has these fields:

- `version`: REQUIRED integer, currently `1`. Missing or unsupported versions fail the plugin.
- `virtual_packages`: REQUIRED object keyed by virtual package name, compared after lowercase normalization.
  Each result MUST be either `null` (explicit absence) or an object containing a REQUIRED `version` string conforming to [CEP 33](./cep-0033.md) and an OPTIONAL `build_string` string conforming to CEP 26, defaulting to `0`.
  The CEP standardizing a name determines its version/build semantics; otherwise the publisher decides and SHOULD use the version field for versions so ordinary constraints work.
- `cache`: OPTIONAL object containing cache hints defined in [Caching](#caching).

Clients MUST ignore unknown top-level keys and unknown keys inside a virtual package result.
Known fields with wrong types are malformed, including negative `ttl_seconds`, non-integer numbers, or strings other than `"REBOOT"` for that field.
Each decoded result version and build string MUST occupy at most 256 UTF-8 bytes.
Each watch list MUST contain at most 32 strings, each at most 4096 UTF-8 bytes after decoding.
Exceeding these limits makes the report malformed, independently of the execution output bound.

The client MUST detect and reject duplicate entries in `virtual_packages`, including names equal after normalization.
It MUST validate the complete report before using any result: every declared name MUST occur, and no undeclared name is allowed.
An omitted name is a contract violation, not an absent capability; `null` expresses absence.
Malformed reports and contract violations fail the entire plugin.

This full contract applies even when some names are overridden or assigned to other registrations.
Only after validation MUST the client discard results for those names.

### Results in the solve

A present result contributes one virtual package record with its name, version and build string; `null` contributes none, subject to the standardized-name requirements below.
Each name has at most one record for the whole solve, shared by packages from every channel, not scoped to the registration's origin.
[CEP 29](./cep-0029.md) matching and ordinary solver behavior apply without new spec syntax or candidate-selection rules.

For a standardized name, a client MUST use an applicable plugin's present result in place of its own detected value.
A client MUST still provide every name CEP 30 and its extensions require: if the plugin reports `null`, fails or is skipped, the client's required value remains.
Standardized meanings and override rules remain unchanged.
A failed or skipped registration MUST NOT remove another registration's result, an override, or a required target-platform client value.

A client MUST NOT attribute plugin detection, client detection or overridden values to a different source in provenance, diagnostics or other retained information.
[User controls](#user-controls) specifies the required plugin attribution.

### Overrides

A client MUST support each nonstandard declared name's [override variable](#registrations):

- A nonempty value MUST be parsed as a version, optionally followed by `=` and a build string; without the latter, the build string is `0`.
- An empty value MUST mean the name is absent.
- An invalid value MUST be an error, not a warning or a fallback to detection.

Overrides for standardized names follow their defining CEPs instead: for example, `CONDA_OVERRIDE_ARCHSPEC` sets the build string, `CONDA_OVERRIDE_UNIX` has no effect, and empty `CONDA_OVERRIDE_CUDA_ARCH` means absence.

An override replaces the entire result for its name, never just the version or build string.
A partially overridden plugin still reports all declared names, but the client MUST discard its overridden results rather than merge them.
A fully overridden registration MUST NOT run.
Overrides apply to names assigned by the registration source, not to shadowed names or alternative registrations.

### Caching

Clients SHOULD cache detection results.
The report's optional `cache` object MAY contain:

| Field | Meaning |
| --- | --- |
| `ttl_seconds` | Nonnegative integer lifetime, or `"REBOOT"` for the current boot session. |
| `watch_paths` | List of absolute paths whose existence or modification time is watched. Relative paths are malformed. |
| `watch_env` | List of environment variable names whose values are watched in the client's own environment. |

A caching client MUST key entries on registration identity, declared names and environment digest, resolving against current repodata before lookup as specified in [Resolution and installation](#resolution-and-installation).
Every entry MUST expire; indefinite caching MUST NOT be expressible.
The following rules apply:

| Condition | Required behavior |
| --- | --- |
| No `ttl_seconds` | Expiry SHOULD default to **1 hour**. |
| Integer lifetime | MUST clamp to at most **30 days**; `0` MUST prevent reuse. |
| `"REBOOT"` | MUST expire at the next reboot or after **30 days**, whichever comes first. The client chooses how to observe reboot boundaries. |
| Reboot cannot be observed | MUST use a short fallback duration, which SHOULD be **1 hour**. |
| Watched path appears, disappears or changes modification time, or a watched variable changes value | MUST expire the entry early, never extend its lifetime. An inaccessible path counts as absent. |

Clients decide whether to retain detector environments between runs, subject to the digest checks.
Cache clearing is specified in [User controls](#user-controls).

### Failure handling

Resolution or installation failure, failed or timed-out activation, missing executable, exceeded bounds, nonzero exit, empty or malformed report, contract violation, and source-defined failure all fail the plugin.
A digest-pin mismatch is one possible source-defined failure.

A failed plugin MUST NOT abort the solve on its own.
The client MUST atomically discard all its results, including individually valid entries.
Only that registration's applicable contribution is removed: overrides, another registration's results, and required client-provided values remain.
Other applicable names are absent, allowing the solver to report unsatisfiable dependencies normally.
The client MUST report every plugin failure and its captured diagnostics, even if the solve succeeds.

### User controls

For each plugin result used, a client MUST record the registration and environment digest that produced it.
On request, it MUST show known registrations, the resolved packages behind each digest, and each plugin's reported results.
When a registration's digest differs from the one last executed, the client MUST report the old and new digests.
A registration source MAY require further approval, and a client MAY show additional information.

A client MUST provide persistent configuration to disable a registration by origin and plugin name.
This CEP does not prescribe a configuration format.
A client MUST also offer an operation to discard cached plugin results and detector environments without clearing unrelated data.

Registration sources define whether digest pins must be offered; any pin facility MUST compare the exact [environment digest](#environment-digest), including dependencies, rather than only a plugin version or build.
A pin mismatch MUST prevent execution and result-cache reuse for that registration and be handled as a plugin failure.

## Security considerations

Detection runs third-party code before the target environment transaction, with the user's privileges.
Dedicated environments and resource bounds are not a sandbox: code can read files, access the network and persist.
Consent covers the plugin and its resolved dependencies, their activation scripts, optional Python byte compilation, invoked vendor tools, and future updates allowed by dependency constraints.
Link scripts are excluded by [Resolution and installation](#resolution-and-installation).

A registration source MUST define consent and how to withhold it; a client MUST NOT run a plugin without that consent.
A client MUST present approval of an origin or plugin name as approval of whatever dependencies resolve now and later, not just the currently named package.
Exact digest pins constrain the resolved records, but do not authenticate artifacts or verify installed files.
A signature requirement covers only the artifacts checked; signing the plugin alone does not attest its dependencies.
Publishers SHOULD treat detection plugins as security-relevant code and SHOULD sign them when supported by the ecosystem's mechanisms, such as [CEP 27](./cep-0027.md).

## Rationale

Executables support library probes, `ioctl`s and vendor tools that a declarative language would need to expose or reimplement.
WASM would likewise require clients to anticipate host APIs and would exclude ordinary shell and Python scripts.
Using the package name as the executable name avoids separate entry-point metadata.

Declared names make the possible solver contribution inspectable before execution; self-enumeration would instead check a plugin against itself and add a second protocol and invocation.
Explicit `null` and complete, atomic reports distinguish missing capabilities from broken detection rather than silently accepting partial output.
A conservative demand scan avoids requiring solvers to await arbitrary work in a mid-solve candidate callback.

The fixed combined output bound limits diagnostics as well as reports; diagnostics need not scale with declared names.
The 30-second default allows slow startup: [PR feedback](https://github.com/conda/ceps/pull/188#discussion_r3896136098) reports Python startup exceeding five seconds on loaded Windows CI.
The 300-second ceiling permits slower systems while bounding the wait for hung detection.
Link scripts would add unbounded installation-time execution; bounded activation provides setup instead.

## Open questions

- **Lockfiles:** [CEP 37](./cep-0037.md) records virtual packages as dependencies, not the values a solve used.
  Recording detection results, registrations and digests belongs in a CEP 37 revision; [User controls](#user-controls) already requires clients to retain that information.
- **Structured explanations:** a future report extension could explain detection decisions without treating logs as protocol data.
- **Client-configured registrations:** a future source could register plugins absent from channel metadata or user-selected builds, defining its own registration form and consent while reusing this protocol.

## Related work

[PEP 817](https://peps.python.org/pep-0817/) proposes executable wheel-variant providers; [PEP 825](https://peps.python.org/pep-0825/) covers the static package format.
A provider defines a property namespace and ranks compatible variants, whereas this CEP supplies only virtual package records for ordinary MatchSpec and solver selection.
Both mechanisms introduce third-party execution and dependency trust; PEP 817 requires a trusted provider version before installation or execution but leaves the trust mechanism to clients.

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

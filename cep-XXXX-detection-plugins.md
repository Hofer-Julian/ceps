# CEP XXXX - Virtual package detection plugins

<table>
<tr><td> Title </td><td> Virtual package detection plugins </td></tr>
<tr><td> Status </td><td> Draft </td></tr>
<tr><td> Author(s) </td><td> Julian Hofer &lt;julian@prefix.dev&gt;, Wolf Vollprecht &lt;wolf@prefix.dev&gt;, Tobias Hunger &lt;tobias@prefix.dev&gt;</td></tr>
<tr><td> Created </td><td> Aug 5, 2026</td></tr>
<tr><td> Updated </td><td> Sep 15, 2026</td></tr>
<tr><td> Discussion </td><td> https://github.com/conda/ceps/pull/188 </td></tr>
<tr><td> Implementation </td><td> https://github.com/conda/rattler/pull/2701 (stack) </td></tr>
<tr><td> Requires </td><td> CEP 26, CEP 29, CEP 30, CEP 32, CEP 33, CEP 34, CEP 46 </td></tr>
</table>

> The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT",
> "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as
> described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) when, and only when, they appear in all capitals, as shown here.

## Abstract

A detection plugin is a conda package with an executable that reports virtual packages as JSON.
Clients install it in a dedicated environment and use its results in the solve.
This CEP defines that protocol independently of the client's implementation language; [CEP XXXX (Channel-provided virtual package plugins)](./cep-XXXX.md) defines discovery through channels.

## Motivation

[CEP 30](./cep-0030.md) standardizes virtual packages detected by clients, but adding detection requires client updates.
Maintainers of external MPI installations, site-specific services and accelerator stacks need a way to distribute their own detectors.

conda's [CEP 4](./cep-0004.md) Python plugin hook supports this, but mamba and Pixi cannot use it.
For example, conda-forge's MPI detector must be installed in conda's own Python environment before solving; a target-environment dependency cannot supply detection for that solve.

## Specification

### Registrations

A **registration** tells the client which detector to run, what virtual packages it reports and where to resolve it.
For example, conda-forge could port its [existing MPI detection](https://github.com/regro/conda-forge-conda-plugins) to a package named `mpi-detect`, reporting `__openmpi` and `__mpich`.

| Field | Meaning | Example value |
| --- | --- | --- |
| Origin | Source-defined identifier | `"https://conda.anaconda.org/conda-forge"` |
| Plugin name | Lowercase normalized package name, also used as the executable name | `"mpi-detect"` |
| Declared names | Non-empty set of virtual package names it reports | `["__openmpi", "__mpich"]` |
| Resolution channels | Ordered, non-empty list of channels for resolving the plugin and its dependencies | `["https://conda.anaconda.org/conda-forge"]` |

The channel's [plugin metadata](./cep-XXXX.md#registration-metadata) gives the plugin name and declared names.
The channel URL and relations determine the other two fields.

Origin and plugin name identify the registration for disabling, pinning and caching.
A **registration source**, such as the channel CEP, supplies these fields.
The source MUST define consent and how users withhold it.
It MUST define collision handling that assigns each name and override variable to at most one registration per solve, and define how invalid declared names are handled without passing them to the client.
Names still assigned to a registration and not [overridden](#overrides) are its **applicable names**.

Declared names MUST satisfy [CEP 26](./cep-0026.md), begin with two underscores, contain at most 64 characters, and match:

```re
^__[a-z0-9][._-]?([a-z0-9]+(\.|-|_|$))*$
```

For example, `__openmpi` is valid; `openmpi` and `__mpi/openmpi` are not.

A name's **override variable** is `CONDA_OVERRIDE_` followed by the name without its two leading underscores, uppercased, with `-` and `.` replaced by `_`.
Thus `__conda-forge_mpi` and `__conda_forge_mpi` both map to `CONDA_OVERRIDE_CONDA_FORGE_MPI` and collide despite being different names.

### The plugin package

A plugin MUST contain an executable named after its normalized package name in a [CEP 32](./cep-0032.md) environment `PATH` directory.
On Windows it MUST have an `.exe`, `.cmd` or `.bat` extension.

A plugin MAY have dependencies, but its direct and transitive virtual-package dependencies MUST be limited to names the client is required to provide under CEP 30 or later virtual-package CEPs.
The plugin and its dependencies MUST NOT rely on pre-link, post-link or pre-unlink scripts.
Plugins SHOULD minimize and pin dependencies.

The plugin MUST be resolvable from its resolution channels for the **host platform**, the machine running the client (CEP 30's native platform).

### Resolution and installation

Clients MUST resolve a [CEP 29](./cep-0029.md) MatchSpec containing only the plugin name, except that a source MAY require a channel qualifier naming a resolution channel.
A source MUST NOT add version or build constraints. Ordinary solver preferences determine the build.

Resolution MUST use only the resolution channels, in their given order, loading their host and `noarch` subdirs.
The only virtual packages available MUST be the client's own CEP 30 virtual packages, honoring `CONDA_OVERRIDE_*`; plugin results MUST NOT participate.

Before execution or result-cache lookup, clients MUST resolve against current repodata and compute the [environment digest](#environment-digest), a fingerprint of all resolved package records.
Clients MUST reuse a detector environment only if its digest matches; otherwise they must install the newly resolved packages.

The **detector environment** contains the plugin and its resolved dependencies.
Installation MUST follow CEP 32 and [CEP 34](./cep-0034.md), with these restrictions:

- The environment MUST be dedicated to detection, separate from the target environment, the client's environment and user working environments. The plugin MUST NOT become a target-environment dependency.
- Clients MUST NOT run any resolved package's pre-link, post-link or pre-unlink scripts.
- Clients MAY skip byte compilation of `noarch: python` packages ([CEP 20](./cep-0020.md)); if performed, it SHOULD be bounded like activation.
- Clients MUST NOT run plugins from incomplete installations and MUST prevent concurrent installations of the same digest from corrupting each other.

Clients MAY share environments between registrations with identical resolved packages.

### Running a plugin

The **target platform** is the platform being solved for.
Clients MUST NOT run plugins when it differs from the host platform.
In that case, plugin-provided names are absent unless overridden, except for values CEP 30 or later virtual-package CEPs require the client to supply for the target.
Clients SHOULD warn once per skipped registration and name its override variables.

Clients MUST NOT run registrations that are disabled, lack required consent or have no applicable names.
Otherwise they use a valid cache entry or execute the plugin.

Clients SHOULD avoid execution when no applicable name can affect the solve.
They MAY conservatively scan fetched records' `depends` and `constrains` fields, including sharded repodata, together with user-requested specs.
They MUST NOT skip a registration whose applicable names the solve could reference, or rely on exact prediction or skipping for correctness.

To execute a plugin, the client MUST:

1. Evaluate the detector environment's activation scripts, including dependencies' scripts, as for normal activation. Activation output is excluded from the report.
2. Prepend the environment's `PATH` directories in CEP 32 order to the inherited `PATH`.
3. Find the normalized plugin executable only in those directories, in that order. On Windows, try `.exe`, `.cmd` and `.bat` in that order within each directory.
4. Start it with no arguments, no input on standard input, and the activated environment. The working directory is unspecified.
5. Capture standard output as the report and standard error separately as diagnostics.

A successful plugin MUST exit with status `0`.

Execution limits:

- The client MUST time out and terminate the process, and SHOULD terminate its descendants. The clock starts at spawn, excluding resolution, installation and activation. The timeout SHOULD default to 30 seconds; clients MAY let users raise it, but MUST NOT exceed 300 seconds. Activation MUST be bounded separately with the same default and ceiling.
- After reading 1 MiB (1,048,576 bytes) of standard output and standard error combined, the client MUST stop reading, terminate a still-running process, and fail the plugin.

Clients MUST retain standard-error diagnostics captured before a bound was reached for [failure reporting](#failure-handling).

### The report

A plugin MUST write exactly one UTF-8 JSON object to standard output, with only whitespace around it.
On a host with Open MPI 5.0.10 but no MPICH, `mpi-detect` could report:

```json
{
  "version": 1,
  "virtual_packages": {
    "__openmpi": { "version": "5.0.10", "build_string": "0" },
    "__mpich": null
  },
  "cache": {
    "ttl_seconds": 86400,
    "watch_paths": ["/opt/openmpi/bin/ompi_info"],
    "watch_env": ["PATH"]
  }
}
```

- `version`: REQUIRED integer, currently `1`. Missing or unsupported versions fail the plugin.
- `virtual_packages`: REQUIRED object keyed by virtual package name, compared after lowercase normalization. Each result MUST be `null` for absence or an object with a REQUIRED `version` string conforming to [CEP 33](./cep-0033.md) and an OPTIONAL `build_string` string conforming to CEP 26, defaulting to `0`.
- `cache`: OPTIONAL object of hints defined in [Caching](#caching).

Standardized names follow their CEP's version and build semantics.
For other names, the publisher decides and SHOULD put versions in the version field so ordinary constraints work.

Clients MUST ignore unknown top-level keys and unknown keys inside a virtual package result.
Known fields with wrong types are malformed.
Each decoded version and build string MUST occupy at most 256 UTF-8 bytes.
Each watch list MUST contain at most 32 strings of at most 4096 UTF-8 bytes each after decoding.
Exceeding these limits makes the report malformed.

Clients MUST reject duplicate `virtual_packages` entries, including names equal after normalization, and MUST validate the complete report before using results.
Every declared name MUST occur, and no undeclared name is allowed.
In the example, `null` reports absent MPICH; omitting `__mpich` would violate the contract.
Malformed or incomplete reports fail the entire plugin.
Only after validation MUST clients discard results for names assigned elsewhere or overridden.

### Results in the solve

A present result contributes one virtual package record with its name, version and build string; `null` contributes none, subject to the standardized-name rules below.
Each name has at most one record for the whole solve, shared across channels.
[CEP 29](./cep-0029.md) matching and ordinary solver behavior apply.

For standardized names, clients MUST replace their detected value with an applicable plugin's present result.
They MUST still supply values required by CEP 30 or later virtual-package CEPs if the plugin reports `null`, fails or is skipped.
A failed or skipped registration MUST NOT remove another registration's result, an override or a required target-platform client value.

### Overrides

Clients MUST support each nonstandard declared name's [override variable](#registrations):

- A nonempty value MUST be parsed as a version, optionally followed by `=` and a build string; the default build string is `0`.
- An empty value MUST mean absence.
- An invalid value MUST be an error, not a warning or a fallback to detection.

Standardized names follow their defining CEPs: `CONDA_OVERRIDE_ARCHSPEC` sets the build string, `CONDA_OVERRIDE_UNIX` has no effect, and empty [`CONDA_OVERRIDE_CUDA_ARCH`](./cep-0046.md) means absence.

An override replaces the whole result for the assigned name, including its build string. It never applies to shadowed names or alternative registrations.

### Caching

Clients SHOULD cache results.
A caching client MUST key entries on registration identity, declared names and environment digest.
The optional `cache` object MAY contain:

| Field | Meaning |
| --- | --- |
| `ttl_seconds` | Nonnegative integer lifetime, or `"REBOOT"` for the current boot session. |
| `watch_paths` | List of absolute paths whose existence or modification time is watched. |
| `watch_env` | List of environment variable names whose values are watched in the client's own environment. |

Relative watch paths, negative or non-integer lifetimes, and lifetime strings other than `"REBOOT"` are malformed.
Every entry MUST expire.

| Condition | Required behavior |
| --- | --- |
| No `ttl_seconds` | Expiry SHOULD default to 1 hour. |
| Integer lifetime | MUST clamp to at most 30 days; `0` MUST prevent reuse. |
| `"REBOOT"` | MUST expire at the next reboot or after 30 days, whichever comes first. Clients choose how to observe reboot boundaries. |
| Reboot cannot be observed | MUST use a short fallback duration, which SHOULD be 1 hour. |
| A watched path appears, disappears or changes modification time, or a watched variable changes value | MUST expire the entry early, never extend its lifetime. Inaccessible paths count as absent. |

#### Environment digest

Clients MUST compute the digest from every resolved package record, including the plugin:

1. Form `<name>\t<version>\t<build>\t<artifact>` for each record, using the normalized lowercase name and verbatim `version` and `build` strings. For `<artifact>`, use non-null `sha256` in lowercase hexadecimal, otherwise non-null `md5` in lowercase hexadecimal, otherwise the file name (`fn`).
2. Sort the lines bytewise ascending.
3. Join with `\n` without a trailing newline, encode as UTF-8, and compute SHA-256, rendered as lowercase hexadecimal.

CEP 26 excludes tabs and newlines from these fields.

### Failure handling

Resolution or installation errors, failed or timed-out activation, missing executable, exceeded bounds, nonzero exit, empty or malformed reports, contract violations and source-defined failures all fail the plugin.

A plugin failure MUST NOT abort the solve on its own.
Clients MUST discard all of that plugin's results atomically, including valid entries, while preserving overrides, other registrations' results and required client-provided values.
Other applicable names are absent, so the solver can report unsatisfiable dependencies normally.
Clients MUST report every plugin failure and its captured diagnostics, even if the solve succeeds.

### User controls

Clients MUST record the registration and environment digest for each plugin result used.
On request, they MUST show known registrations, the resolved packages behind each digest and each plugin's reported results.
They MUST NOT misattribute plugin detection, client detection or overrides in any retained information or diagnostics.
When a registration's digest differs from the one last executed, clients MUST report the old and new digests.
A source MAY require further approval.

Clients MUST provide persistent configuration to disable registrations by origin and plugin name; its format is client-defined.
They MUST also offer an operation to discard cached plugin results and detector environments without clearing unrelated data.

Sources define whether clients must offer digest pins. Any pin facility MUST compare the exact [environment digest](#environment-digest), including dependencies.
A mismatch MUST prevent execution and result-cache reuse and be handled as a plugin failure.

## Backwards compatibility

The protocol uses ordinary conda packages and virtual package records; it changes neither package metadata nor MatchSpec syntax.

## Security considerations

Detection runs third-party code before the target environment transaction, with the user's privileges.
Dedicated environments and resource bounds are not a sandbox: plugins can read files, access the network and persist.
Consent covers resolved dependencies, activation scripts, optional Python byte compilation, invoked vendor tools and future updates allowed by dependency constraints.

Clients MUST present approval of an origin or plugin as approval of its dependencies now and in future resolutions.
Digest pins constrain resolved records without authenticating artifacts or verifying installed files.
Signing a plugin alone does not attest its dependencies.
Publishers SHOULD treat plugins as security-relevant code and SHOULD sign them where supported, such as through [CEP 27](./cep-0027.md).

## Rationale

Executables can probe libraries, make `ioctl` calls and invoke vendor tools.
A declarative language or WASM interface would require clients to anticipate those host APIs; executables also support ordinary shell and Python scripts.

Declaring names makes a plugin's possible results inspectable before execution.
Self-enumeration would add a second invocation that checks the plugin against itself.

The 30-second default accommodates [slow Python startup on Windows CI](https://github.com/conda/ceps/pull/188#discussion_r3896136098); the 300-second ceiling bounds the wait for hung detection.
Link scripts would add unbounded installation-time execution; bounded activation supplies setup instead.

## Copyright

All CEPs are explicitly [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).

# CEP XXXX - Channel Registration of Virtual Package Detectors

<table>
<tr><td> Title </td><td> Channel Registration of Virtual Package Detectors </td></tr>
<tr><td> Status </td><td> Draft </td></tr>
<tr><td> Author(s) </td><td> Julian Hofer &lt;julian@prefix.dev&gt;, Wolf Vollprecht &lt;wolf@prefix.dev&gt;, Tobias Hunger &lt;tobias@prefix.dev&gt;</td></tr>
<tr><td> Created </td><td> Aug 5, 2026</td></tr>
<tr><td> Updated </td><td> Sep 16, 2026</td></tr>
<tr><td> Discussion </td><td> https://github.com/conda/ceps/pull/188 </td></tr>
<tr><td> Implementation </td><td> https://github.com/conda/rattler/pull/2701 (stack) </td></tr>
<tr><td> Requires </td><td> CEP XXXX (Virtual Package Detector Protocol), CEP 16, CEP 26, CEP 30, CEP 36, CEP 42 </td></tr>
</table>

> The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT",
> "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as
> described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) when, and only when, they appear in all capitals, as shown here.

## Abstract

This CEP lets channels register virtual package detectors in `repodata.json`.
Clients resolve the detectors from the registering channel and its related channels, then use the detected virtual packages in the solve.
The [detector protocol](./cep-XXXX-virtual-package-detector-protocol.md) defines execution; this CEP adds channel discovery, priority and consent.

## Motivation

The conda-forge external MPI packages require users to install their detector into conda before creating an environment.
Channel registrations remove that setup step: adding the channel also makes its detection available.

## Specification

Channels and subdirs follow [CEP 26](./cep-0026.md); package-name comparisons MUST use lowercase normalized names.
Repodata follows [CEP 36](./cep-0036.md), and channel relations and priority follow [CEP 42](./cep-0042.md).
A channel's registrations participate whenever the channel does, whether configured directly or loaded through relations.

### Registration metadata

A channel MAY include `virtual_package_detectors` in the `info` dictionary of `repodata.json`.
For example, conda-forge could register a proposed `mpi-detect` package to report the names used by its [existing MPI detector](https://github.com/regro/conda-forge-conda-plugins):

```json
{
  "info": {
    "subdir": "linux-64",
    "virtual_package_detectors": {
      "mpi-detect": ["__openmpi", "__mpich"]
    }
  }
}
```

Each key names both the detector package and its [executable](./cep-XXXX-virtual-package-detector-protocol.md#the-detector-package); its array lists the virtual packages to report.

A client MUST read the field as an opaque JSON value before validating it, so invalid registrations do not prevent parsing the rest of the repodata.
An absent field or empty dictionary registers no detectors. Otherwise:

- The value MUST be a dictionary mapping detector package names to arrays of virtual package names. `null` and wrong-shaped values are registration errors.
- Each key MUST be a valid installable package name, not a virtual package name beginning with `__`, and MUST name a package served by the declaring channel in one of its subdirs. An invalid key is a registration error; failure to resolve a syntactically valid name is a [detector failure](./cep-XXXX-virtual-package-detector-protocol.md#failure-handling).
- Each array MUST contain 1 to 16 entries before invalid names are dropped. Names MUST satisfy the detector protocol's [name rules](./cep-XXXX-virtual-package-detector-protocol.md#registrations).
- Clients MUST drop invalid virtual package names, SHOULD report them, and MUST ignore detectors left with no valid names. The remaining names are the registration's **virtual package names**.

#### Loaded subdirs and limits

The **loaded subdirs** are those fetched from a channel for the solve, normally the target platform's subdir and `noarch` as recommended by CEP 42.
Clients MUST combine registrations from all loaded subdirs and MUST NOT include unloaded subdirs.
Entries with the same normalized detector name form one registration containing the union of their names.
A channel MAY register a detector only in relevant platform subdirs; a `noarch` registration is available across platforms.

The following are registration errors across the loaded subdirs:

- An array outside the 1 to 16 entry limit, or a detector's union exceeding 16 names, counted before invalid names are dropped.
- More than 64 distinct detectors, counted before empty registrations are removed.
- Two detectors declaring the same normalized virtual package name, or a duplicate normalized name within one array. Repeating a name for the same detector in different subdirs is allowed.
- Distinct virtual package names mapping to the same [override variable](./cep-XXXX-virtual-package-detector-protocol.md#registrations).
- Duplicate normalized detector keys within one subdir's dictionary. Clients whose JSON parser cannot expose duplicate keys need not detect them.

Clients MUST report any registration error and ignore the channel's entire registration set across all loaded subdirs.
They MUST NOT reject the surrounding repodata or abort the solve.
Individually invalid virtual package names are the exception: they are dropped as specified above.

#### Sharded repodata

For [CEP 16](./cep-0016.md) sharded repodata, `virtual_package_detectors` MAY appear in the shard index's `info` dictionary with the same schema and semantics.
A channel serving both forms MUST publish consistent registrations. Clients need only read the form they loaded.

### Names and channel priority

A virtual package name has one meaning throughout a solve.
For names not standardized by a CEP, channels SHOULD include their channel name, for example `__conda_forge_mpi_abi` rather than `__mpi_abi`.
The MPI examples retain the names used by the existing conda detector.
Channels SHOULD register standardized names only to replace client detection, following the detector protocol's [standardized-name rules](./cep-XXXX-virtual-package-detector-protocol.md#results-in-the-solve).

Different channels MAY register the same name.
The first channel in CEP 42's resolved order MUST win for both identical normalized names and distinct names mapping to the same override variable.
The lower-priority name is **shadowed**. A channel reached through multiple relation paths counts once.
Disabling or failing the winner MUST NOT enable a shadowed registration as a fallback.

A fully shadowed registration MUST NOT run.
Clients SHOULD report skipped registrations and the winning channel for each shadowed name.
A partially shadowed registration remains eligible: clients MUST validate its complete report against all virtual package names before discarding shadowed results, as specified in [The report](./cep-XXXX-virtual-package-detector-protocol.md#the-report).
Overrides apply only to the winning name and registration.

### Resolution and participation

A registration's **origin** is the registering channel's CEP 26 base URL.
Its **resolution channels** are the ordered channels CEP 42 would resolve with that channel as the only user-specified channel, using relations from its loaded subdirs and the client's usual depth limit.
If a cycle or depth limit prevents relation resolution, clients MUST use the registering channel alone and SHOULD warn.

The detector's MatchSpec MUST be qualified with the registering channel.
Its dependencies MUST resolve only from the resolution channels, excluding unrelated user-configured channels.
A related channel's newer, same-named package therefore cannot replace the registered detector.

Clients MUST run participating detectors subject to shadowing and the detector protocol's [execution rules](./cep-XXXX-virtual-package-detector-protocol.md#running-a-detector), then apply its [solver integration rules](./cep-XXXX-virtual-package-detector-protocol.md#results-in-the-solve).

### Consent and user controls

Configuring a channel consents to running its registered detectors, including those of channels loaded through its relations.
Clients MAY require additional opt-in and MUST NOT run detectors from channels outside the solve.

Clients MUST provide persistent configuration to:

- Disable an origin or an individual registration identified by origin and detector name.
- Pin a registration to an exact [environment digest](./cep-XXXX-virtual-package-detector-protocol.md#environment-digest). A mismatch MUST fail the detector and report both digests.
- Override a detector-provided name or declare it absent, alongside the detector protocol's [environment-variable overrides](./cep-XXXX-virtual-package-detector-protocol.md#overrides).

Clients MUST also provide the detector protocol's [reporting and cache-clearing controls](./cep-XXXX-virtual-package-detector-protocol.md#user-controls).

## Example

Suppose conda-forge registers `mpi-detect` as above and adds `__openmpi >=5.0,<6.0a0` to an external Open MPI build's dependencies.
When solving for that build on the host platform, the client installs `mpi-detect` and its dependencies from conda-forge in a detector environment.
The detector finds `/opt/openmpi/bin/ompi_info` on `PATH`, identifies Open MPI 5.0.10, and reports:

```json
{
  "version": 1,
  "virtual_packages": {
    "__openmpi": { "version": "5.0.10" },
    "__mpich": null
  }
}
```

The `__openmpi 5.0.10` record satisfies the dependency through ordinary MatchSpec matching.
Without Open MPI, the detector reports `null` for `__openmpi` too, leaving that external build's dependency unsatisfiable.
Packages from other channels, including those that load conda-forge through a relation, use the same result.

## Backwards compatibility

The optional `info.virtual_package_detectors` field requires no `repodata_version` change.
Under CEP 36, clients SHOULD ignore unrecognized `info` keys; clients without detector support cannot satisfy dependencies on names they do not otherwise provide.
Channels without registrations are unaffected, and detector packages need no new `index.json` fields.

## Security considerations

Channel detectors run before the solve completes, even if none of the channel's packages is ultimately installed.
Trust extends to dependencies from related channels and future updates.
The detector protocol's [security considerations](./cep-XXXX-virtual-package-detector-protocol.md#security-considerations) also apply.

Channels SHOULD treat detectors as security-relevant code, sign them where supported by mechanisms such as [CEP 27](./cep-0027.md), and keep dependencies few and pinned.
They SHOULD register detectors only in their intended subdirs.

## Rationale

Putting registrations in repodata's `info` dictionary follows CEP 42 and avoids a separate request, cache and schema for usually absent metadata.

Keying the map by detector lets one process report several names without clients regrouping a name-to-detector map.
Channels control detector versions through the builds they serve, removing older builds or moving them to a label when needed.

MatchSpecs match one name across the solve. Channel-specific names inside the solver would require dependency rewriting; distinctive public names and channel priority avoid this.

## Copyright

All CEPs are explicitly [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).

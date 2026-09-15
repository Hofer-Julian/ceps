# CEP XXXX - Channel-provided virtual package plugins

<table>
<tr><td> Title </td><td> Channel-provided virtual package plugins </td></tr>
<tr><td> Status </td><td> Draft </td></tr>
<tr><td> Author(s) </td><td> Julian Hofer &lt;julian@prefix.dev&gt;, Wolf Vollprecht &lt;wolf@prefix.dev&gt;, Tobias Hunger &lt;tobias@prefix.dev&gt;</td></tr>
<tr><td> Created </td><td> Aug 5, 2026</td></tr>
<tr><td> Updated </td><td> Sep 14, 2026</td></tr>
<tr><td> Discussion </td><td> https://github.com/conda/ceps/pull/188 </td></tr>
<tr><td> Implementation </td><td> https://github.com/conda/rattler/pull/2701 (stack) </td></tr>
<tr><td> Requires </td><td> CEP XXXX (Virtual package detection plugins), CEP 16, CEP 26, CEP 30, CEP 36, CEP 42 </td></tr>
</table>

> The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT",
> "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as
> described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) when, and only when, they appear in all capitals, as shown here.

## Abstract

This CEP extends [CEP XXXX (Virtual package detection plugins)](./cep-XXXX-detection-plugins.md), below "the plugin CEP", with channel-provided registrations.
A channel declares detection plugins in `repodata.json`; clients resolve them from that channel and its relations and use their results in the solve.
The plugin CEP defines the execution protocol. This CEP defines discovery, channel priority and consent.

## Motivation

Adding a channel should be enough to make its packages installable. The conda-forge external MPI packages currently require users to install a client plugin separately, a setup step they must know about in advance.
Channel registrations let package builders ship detection alongside the packages that need it, without duplicating detection in each client or requiring per-user overrides.

## Specification

Channels, subdirs and normalized package names follow [CEP 26](./cep-0026.md); package-name comparisons MUST use lowercase normalized names.
Repodata follows [CEP 36](./cep-0036.md), and channel relations and resolved channel order follow [CEP 42](./cep-0042.md).

### Registration metadata

A channel MAY include `virtual_package_plugins` in the `info` dictionary of `repodata.json`:

```json
{
  "info": {
    "subdir": "linux-64",
    "virtual_package_plugins": {
      "rocm-detect": ["__acme_rocm"],
      "cuda-detect": ["__acme_cuda", "__acme_cuda_arch"]
    }
  }
}
```

A client MUST read the field as an opaque JSON value before validating it, so malformed registrations do not prevent parsing the rest of the document.
An absent field or empty dictionary registers no plugins. Any other value MUST be a dictionary mapping plugin package names to non-empty arrays of virtual package names; `null` and wrong-shaped values are registration errors.

- Each key MUST be a valid installable package name, not a virtual package name beginning with `__`, and MUST name a package the declaring channel serves in one of its subdirs. An invalid plugin name is a registration error. A syntactically valid name that cannot be resolved instead fails under the plugin CEP's [failure handling](./cep-XXXX-detection-plugins.md#failure-handling).
- The key identifies both the plugin package and its executable, as specified in [The plugin package](./cep-XXXX-detection-plugins.md#the-plugin-package). One plugin MAY provide several virtual packages.
- Each array MUST contain between 1 and 16 entries, counted before dropping invalid names. Every name MUST satisfy the plugin CEP's [virtual package name rules](./cep-XXXX-detection-plugins.md#registrations).
- An invalid virtual package name MUST be dropped without invalidating the other names or registrations. Clients SHOULD report discarded names and MUST ignore plugins left with no valid names. The remaining names are the registration's **declared names**.

#### Loaded subdirs and limits

The **loaded subdirs** are the subdirs a client fetched from a channel for the solve. CEP 42 recommends the target platform's subdir and `noarch`; any others a client loads participate on the same terms.
A client MUST combine registrations across the channel's loaded subdirs, and MUST NOT include registrations from unloaded subdirs.
The same normalized plugin name in different subdirs forms one registration whose names are the union of its arrays.
For example, `linux-64` declaring `x` for `["__a"]` and `noarch` declaring `x` for `["__a", "__b"]` yields one registration for `__a` and `__b`.

The following are registration errors, evaluated across the loaded subdirs:

- An array outside the 1 to 16 entry limit, or a plugin's union containing more than 16 names, counted before invalid names are dropped.
- More than 64 distinct plugins registered by one channel, counted before empty registrations are removed.
- Two plugins declaring the same normalized virtual package name, or one array containing the same normalized virtual package name more than once. Repeating a name for the same plugin across different subdirs is allowed.
- Two distinct declared names mapping to the same `CONDA_OVERRIDE_*` variable under the plugin CEP's [override mapping](./cep-XXXX-detection-plugins.md#overrides).
- Duplicate normalized plugin keys in a subdir's registration dictionary. A client whose JSON parser cannot expose duplicate keys is not required to detect those duplicate keys.

Registrations within a channel have no ordering. A client MUST NOT choose between colliding registrations or silently merge them.
For any registration error, it MUST report the error and ignore the channel's entire registration set across all loaded subdirs. It MUST NOT reject the surrounding repodata or abort the solve.
This whole-set rule does not apply to individually invalid virtual package names, which are dropped as specified above.

A channel MAY register a plugin only in relevant platform subdirs; registering it in `noarch` makes it available across platforms.

#### Sharded repodata

For [CEP 16](./cep-0016.md) sharded repodata, `virtual_package_plugins` MAY appear in the shard index's `info` dictionary with the same schema and semantics.
A channel serving both forms MUST publish consistent registrations. Clients read the form they loaded and need not fetch the other to check consistency.

### Names and channel priority

A name has one meaning throughout a solve. For names not standardized by a CEP, channels SHOULD include their channel name, for example `__acme_rocm` rather than `__rocm`.
Standardized names retain their specified meanings; channels SHOULD register plugins for them only when intending to replace client detection, subject to the plugin CEP's [rules for standardized names](./cep-XXXX-detection-plugins.md#results-in-the-solve).
Channel priority selects a provider, but cannot reconcile channels using a name for different capabilities.

Different channels MAY register the same name. The first channel in CEP 42's resolved channel order MUST win when registrations declare either the same normalized name or distinct names mapping to the same override variable.
The lower-priority name is shadowed in either case. A channel reached through multiple relation paths counts once.
Priority is determined before execution: disabling or failing the winning plugin MUST NOT enable a shadowed registration as a fallback.

- A registration whose names are all shadowed MUST NOT run. Clients SHOULD report skipped registrations and the winning channel for each shadowed name.
- A partially shadowed registration remains eligible for execution. Clients MUST validate its entire report against all its declared names, then discard results for shadowed names, as specified in [The report](./cep-XXXX-detection-plugins.md#the-report).
- Overrides apply to the winning name and registration, never to a shadowed registration or name. See [Overrides](./cep-XXXX-detection-plugins.md#overrides).

At most one registration can therefore supply each name or own each override variable. A plugin's failure or a nonhost-target skip MUST NOT remove another winning plugin's results or values the client is required to supply for the target; see [Failure handling](./cep-XXXX-detection-plugins.md#failure-handling) and [Results in the solve](./cep-XXXX-detection-plugins.md#results-in-the-solve).

### Resolution and participation

For the plugin CEP's [registration contract](./cep-XXXX-detection-plugins.md#registrations), a registration's **origin** is the registering channel's CEP 26 base URL.
Its **resolution channels** are the ordered channels CEP 42 would resolve if the registering channel were the only user-specified channel, using relations from its loaded subdirs and the client's usual depth limit.
If relation resolution fails because of a cycle or the depth limit, the resolution channels MUST be the registering channel alone, and the client SHOULD warn.

The plugin's MatchSpec MUST be qualified with the registering channel. Its dependencies MUST resolve only from the resolution channels, never from unrelated channels the user also configured.
This preserves the registering channel's choice of detector rather than allowing a related channel's newer, same-named package to replace it.
Installation and execution follow the plugin CEP.

Registrations participate whenever their channel participates, whether directly configured or included through CEP 42 relations.
A client MUST run participating plugins subject to the plugin CEP's [execution eligibility rules](./cep-XXXX-detection-plugins.md#running-a-plugin), including disabling, overrides, optional demand scanning and the host/target restriction, and to shadowing above.
Results apply globally: every package in the solve sees the same virtual package records, not a channel-specific view.

### Consent and user controls

Configuring a channel, directly or through a configured channel's relations, consents to running its registered plugins.
A client MUST NOT run a plugin from a channel that does not participate in the solve.
This CEP requires no additional opt-in, but clients MAY require one.

Clients MUST provide persistent configuration to:

- Disable all registrations from an origin, as well as individual registrations identified by origin and plugin name under [User controls](./cep-XXXX-detection-plugins.md#user-controls).
- Pin a registration to an exact [environment digest](./cep-XXXX-detection-plugins.md#environment-digest). A resolved digest mismatch MUST fail the plugin and report both digests. The digest fingerprints the plugin's resolved package records and dependencies; it is not proof of authenticity or unchanged on-disk code.
- Override a plugin-provided name or declare it absent, alongside the `CONDA_OVERRIDE_*` environment variables specified in [Overrides](./cep-XXXX-detection-plugins.md#overrides). This CEP does not prescribe a configuration format.

Clients MUST also provide the plugin CEP's [cache-clearing controls](./cep-XXXX-detection-plugins.md#caching) and [execution, provenance and digest-change reporting](./cep-XXXX-detection-plugins.md#user-controls). Cache clearing is an operation, not a required persistent setting.

## Example

`https://example.org/acme` ships `rocm-detect` and packages that need ROCm. Its `linux-64/repodata.json` contains the following entries, with unrelated metadata omitted:

```json
{
  "info": {
    "subdir": "linux-64",
    "virtual_package_plugins": { "rocm-detect": ["__acme_rocm"] }
  },
  "packages.conda": {
    "rocm-detect-1.0.0-h1234567_0.conda": {
      "name": "rocm-detect", "version": "1.0.0"
    },
    "mytool-2.0.0-h1234567_0.conda": {
      "name": "mytool", "version": "2.0.0", "depends": ["__acme_rocm >=6.0"]
    }
  }
}
```

When solving for `mytool` on the host platform, the client installs `rocm-detect` and its resolved dependencies in a detector environment and runs the `rocm-detect` executable. It reports:

```json
{
  "version": 1,
  "virtual_packages": { "__acme_rocm": { "version": "6.2.1" } },
  "cache": { "ttl_seconds": 86400, "watch_paths": ["/sys/module/amdgpu/version"] }
}
```

The resulting `__acme_rocm 6.2.1` record satisfies `mytool`'s dependency through ordinary MatchSpec matching.
Without ROCm, the plugin reports `{"version": 1, "virtual_packages": {"__acme_rocm": null}}`; the dependency is unsatisfiable instead of allowing installation of a package that cannot run.
A package from a channel that includes `acme` through a CEP 42 relation uses the same result.

## Compatibility

The additive `info.virtual_package_plugins` field does not bump `repodata_version`. Under CEP 36, unsupported clients SHOULD ignore unrecognized `info` keys and retain their existing behavior; they cannot satisfy dependencies on virtual packages they do not otherwise provide.
Channels without registrations and existing packages are unchanged. Plugins are ordinary packages and need no new `index.json` fields.
Standardized virtual packages retain their meanings and the client's obligations under their CEPs.

## Security considerations

Channel configuration permits detection code to run after detector installation but before the solve completes and before the target environment transaction.
Ordinary channel packages can already execute code through installation and activation; plugins change when code runs and can run even when none of the channel's packages is ultimately installed.
Trust includes the plugin and its resolved dependencies, including packages from related channels and future updates. The plugin CEP's [security considerations](./cep-XXXX-detection-plugins.md#security-considerations) apply: a detector environment and execution bounds are not a sandbox.

Disabling registrations and digest pinning limit this consent. Clients may require additional opt-in, including for relation-loaded channels, but this CEP does not require it.
Where clients perform the optional demand scan, only potentially relevant plugins run; without it, participating registrations may run more broadly under the plugin CEP's rules.

Channels SHOULD:

- Treat detection plugins as security-relevant code and sign them where ecosystem mechanisms permit, such as [CEP 27](./cep-0027.md).
- Register plugins with few dependencies and pin those dependencies to stabilize the resolved packages between runs.
- Register plugins only in their intended subdirs so other platforms do not need to skip them.

## Rationale

- **Repodata rather than a separate file:** `info` follows CEP 42 and avoids an extra request and separate caching, validation and versioning for usually absent metadata.
- **Plugin-keyed map:** one process can report several names. An inverse name-to-plugin map would require regrouping to avoid duplicate execution.
- **Bare plugin names:** the key also identifies the executable. Channels control updates through the builds they serve and can remove older builds or move them to a label; registration-level version constraints are unnecessary.
- **Global names:** MatchSpecs and solvers match one name across the solve. Per-channel views would require dependency rewriting and internal names throughout clients. Distinctive names and existing channel priority avoid that machinery.
- **Loaded subdirs only:** fetching every subdir adds requests and registrations irrelevant to the solve. `noarch` already provides cross-platform registration.
- **Separate CEPs:** plugin authors need the execution protocol; channel maintainers need discovery and consent rules. Other registration sources can reuse the protocol without adopting channel consent.

## Open questions

- Should channels validate at upload time that every registration names a package they serve?
- Should stronger name-ownership enforcement supplement naming recommendations and shadowing diagnostics, such as reserved-prefix checks or a registry of channel-introduced names?
- An independently configured registration source, including user-selected plugins not registered by a channel, remains deferred in the plugin CEP's [open questions](./cep-XXXX-detection-plugins.md#open-questions).

## References

- [CEP XXXX - Virtual package detection plugins](./cep-XXXX-detection-plugins.md)
- [CEP 16 - Sharded Repodata](./cep-0016.md)
- [CEP 26 - Identifying Packages and Channels in the conda Ecosystem](./cep-0026.md)
- [CEP 27 - Standardizing a publish attestation for the conda ecosystem](./cep-0027.md)
- [CEP 29 - The `MatchSpec` query language](./cep-0029.md)
- [CEP 30 - Virtual packages](./cep-0030.md)
- [CEP 34 - Contents of conda packages](./cep-0034.md)
- [CEP 36 - Package metadata files served by conda channels](./cep-0036.md)
- [CEP 42 - Channel relations in repodata](./cep-0042.md)

## Copyright

All CEPs are explicitly [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).

# CEP XXXX - Channel-provided virtual package plugins

<table>
<tr><td> Title </td><td> Channel-provided virtual package plugins </td></tr>
<tr><td> Status </td><td> Draft </td></tr>
<tr><td> Author(s) </td><td> Wolf Vollprecht &lt;wolf@prefix.dev&gt;<br/>Tobias Hunger &lt;tobias@prefix.dev&gt;</td></tr>
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

[CEP XXXX (Virtual package detection plugins)](./cep-XXXX-detection-plugins.md), below "the plugin CEP", defines detection plugins and how clients run them, but not how clients discover them.
This CEP defines the first registration source: a channel declares, in the `info` dictionary of its `repodata.json`, that a package it serves is a detection plugin for one or more virtual packages.
A client resolving that channel runs the plugin as specified by the plugin CEP and uses its reported values in the solve like client-detected virtual packages.

A plugin answers only for names its channel advertised, its dependencies come only from that channel and the channels it relates to, and its verdicts can be overridden or suppressed from the environment.
Virtual package names are assumed to be unique across channels, and a channel introducing one SHOULD build its own name into it.
Configuring a channel consents to running its plugins; see [Security considerations](#security-considerations).

## Motivation

The plugin CEP explains why detection needs to be extensible. Channels provide the detection code because:

- Package builders know how to detect the capabilities their packages depend on, typically by wrapping a vendor tool in a short script. They already ship conda packages.
- Adding a channel should be enough to make its packages installable. The conda-forge external MPI packages currently require users to install a plugin separately, a step users need to know about in advance.
- Maintaining detection in each client or each user's configuration duplicates that knowledge.

Current workarounds include setting `CONDA_OVERRIDE_*` variables by hand in every environment, accepting runtime failures instead of solve-time errors, writing wrappers that solve twice, and maintaining vendor-specific detection in clients whose authors cannot test the hardware.

## Specification

The terms conda channel, and channel subdirectory (subdir) MUST be understood as specified by [CEP 26](./cep-0026.md).
The `repodata.json` schema is defined by [CEP 36](./cep-0036.md).
Channel relations and the resolved channel order are defined by [CEP 42](./cep-0042.md).
Registration, origin, declared names, resolution channels, live registration, closure and digest are used as the plugin CEP defines them.
Package names are compared in their normalized form, which per CEP 26 is the lowercase form.

The **loaded subdirs** of a channel, for a solve, are the subdirs the client fetched from that channel for the solve.
CEP 42 recommends loading the target platform's subdir and `noarch`. This CEP assumes those subdirs; a client that loads others treats their registrations the same way.

### Registering plugins in `info`

A `repodata.json` file MAY include a `virtual_package_plugins` key in its `info` dictionary.
If present, it MUST be a dictionary mapping a **package name** to a **non-empty array of virtual package names**:

```json
{
  "info": {
    "subdir": "linux-64",
    "virtual_package_plugins": {
      "rocm-detect": ["__rocm"],
      "cuda-detect": ["__cuda", "__cuda_arch"]
    }
  }
}
```

A client MUST first read `virtual_package_plugins` as an opaque JSON value, then validate it. This lets the client report and drop an invalid value without disrupting the rest of the document (see [Errors](#errors)).

- Each key MUST be the name of a package the declaring channel serves in one of its subdirs.
  This is the registration's plugin name, which the plugin CEP also uses as the executable name.
  It MUST therefore be a valid *package* name and MUST NOT be a virtual package name: a name with the `__` prefix cannot identify an installable package.
  An invalid package name is an error, and a client MUST treat it as such rather than ignoring the entry. The key identifies code to run, so an invalid key makes the channel's request unclear.
  A valid package name that the channel does not serve cannot be detected at this stage; it fails during resolution, as the plugin CEP describes.
- Each value MUST be the array of virtual package names that plugin speaks for.
  One plugin MAY speak for several virtual package names.
  An expensive query can return several values in one run, rather than several plugins repeating the query and discarding unused results.
- Every name in every array MUST be a valid virtual package name as the plugin CEP defines it, and SHOULD contain the channel name; see [Naming](#naming).
  A name that is not valid MUST be dropped; otherwise it would enter the solve as an unusable dependency specification.
  Dropping it MUST NOT invalidate the other names of the same plugin, and MUST NOT invalidate the rest of the `repodata.json`: a client MUST parse registrations leniently and discard only what is invalid.
  A client SHOULD report discarded names so that a channel maintainer can find the error.
  Plugins left with no valid name MUST be ignored.
- One array MUST hold between 1 and 16 (inclusive on both ends) names, counted before any invalid name is dropped, and the union of a plugin's arrays across the loaded subdirs MUST hold at most 16 names, counted the same way.
  An array or union outside that range is an error.
- One channel MUST NOT register more than 64 plugins, counted over the loaded subdirs.
  This bounds the environments and processes one channel can request, alongside the plugin CEP's demand scan where a client performs it.
  Exceeding this limit is an error.
- Additional keys in `info` beyond those CEP 36 and its extensions define SHOULD, per CEP 36, be ignored by clients that do not recognize them.
  Clients that do not implement this CEP ignore `virtual_package_plugins` and keep their existing behavior.
  CEP 36 also says such keys SHOULD NOT be present; this CEP, like CEP 42, is an extension that defines one.
- If `virtual_package_plugins` is absent or empty, the channel registers no plugins.
- A `virtual_package_plugins` that is present but is not a map of registrations, including an explicit `null`, is an error.
  A client MUST NOT treat such a value as an empty registration set. An absent field already represents that case.

#### Subdirs

Declarations are per subdir, like CEP 42's `channel_relations`.
A client MUST treat the registrations found in a channel's loaded subdirs as a single set.
A plugin name appearing in multiple loaded subdirs forms one registration, with the union of its declared-name arrays. For example, `linux-64` registering `x` for `["__a"]` and `noarch` registering `x` for `["__a", "__b"]` yields one registration of `x` for `__a` and `__b`.
Registrations in subdirs not loaded for the solve do not participate.
A channel MAY register a plugin in only some subdirs to restrict it to relevant platforms. Registering it in `noarch` makes it available everywhere.

Over the union, a channel MUST NOT register two plugins for the same (normalized) virtual package name, MUST NOT let one plugin register the same (normalized) virtual package name twice within one array, and MUST NOT register two names that map to the same override variable of the plugin CEP.
Within one array of one subdir a channel MUST NOT list the same (normalized) package name under two keys; a client whose JSON parser cannot see duplicate keys is not required to detect this.
Registrations that are each valid on their own MAY still collide once merged.
Registrations within a channel have no ordering. A client encountering a collision MUST treat it as an error, and MUST NOT pick one of the colliding registrations or silently merge them.

The declared names of a registration are the names left after invalid ones are dropped.

#### Errors

Where this section defines a registration error, a client MUST report it to the user and MUST then ignore the channel's entire `virtual_package_plugins` set across all loaded subdirs, as though the channel had registered no plugins.
A client MUST NOT reject the surrounding `repodata.json` and MUST NOT abort the solve: malformed registrations do not prevent use of the channel's packages.

These errors leave the intended registration set unclear. Ignoring the whole set avoids running an arbitrary subset.
An invalid *name* is different: it can be dropped without affecting the remaining registrations.

#### Sharded repodata

When a channel serves sharded repodata as defined by [CEP 16](./cep-0016.md), the `virtual_package_plugins` field MAY also appear in the `info` dictionary of the shard index.
Its schema and semantics there are identical.
If a channel serves both `repodata.json` and sharded repodata, the registrations declared in both MUST be consistent.
A client reads whichever of the two it loaded and is not expected to check the other.

### Naming

A virtual package name has one meaning per solve and is shared across the ecosystem.
Channel priority selects the plugin for a contested name (see [Contested names](#contested-names)), but cannot reconcile channels that use the name for different capabilities.

A channel registering a plugin for a name that no CEP standardizes SHOULD include its channel name: for example, `__acme_rocm` rather than `__rocm` for a channel named `acme`.
The resulting name MUST still satisfy the plugin CEP's rules.

The registering channel also builds the packages that depend on the name. Other channels need not know it, and including the channel name does not make dependencies harder to specify.
Distinctive names reduce the need for coordination between unrelated channels and prevent one channel's hardware detection from replacing another's because they chose the same name.

Names standardized by CEP 30 or a later CEP are intentionally shared. A channel registering a plugin for one offers its own detection for that capability.
Channels SHOULD NOT do so unless they intend to replace the client's own detection, which the plugin CEP allows but never lets remove the name.

### Contested names

Two different channels MAY register different plugins for the same name.
The channel that comes first in the solve's resolved channel order, produced by CEP 42 from the user's channels, wins the name and shadows every other registration for it.
A channel reached by more than one relation path is one channel, as it is for CEP 42.

Shadowing chooses the plugin, not the meaning of the name. A client cannot tell whether the channels meant the same capability; see [Naming](#naming).

- A shadowed name is not wanted, in the plugin CEP's sense, for the registration that lost it.
  A registration all of whose names are shadowed is therefore not live and MUST NOT be run.
  A client SHOULD report that the registration was skipped and which channel took each name, rather than silently omitting it.
- A registration shadowed for only some of its names is run if it is otherwise live, held to the full contract as the plugin CEP requires, and its verdicts for shadowed names are discarded.
- An override applies to the name, so it reaches whichever registration won the name; a registration that lost a name never sees an override for it.

After shadowing, at most one registration answers for each name, as the plugin CEP requires of a registration source.

### Resolution channels

The resolution channels of a registration are the channels CEP 42's resolution would produce if the registering channel were the only user-specified channel, using the `channel_relations` of that channel's loaded subdirs and the same depth limit the client applies elsewhere.
If that resolution fails, because of a cycle or the depth limit, the resolution channels are the registering channel alone, and the client SHOULD warn.

The plugin package itself MUST be resolved from the registering channel: the MatchSpec the plugin CEP prescribes carries the registering channel as its channel qualifier.
Its dependencies come from the resolution channels.
A client MUST NOT resolve either from any other channel the user happens to have configured.

The registering channel and its relations determine the plugin's supply chain. Using the user's full channel list would allow a registration to pull code from unrelated channels.

The origin of a registration is the registering channel's base URL as CEP 26 defines it.

### Which registrations take part in a solve

A registration participates when its channel does, whether configured by the user or included through a configured channel's CEP 42 relations.
Relations determine which registrations participate, but do not scope their verdicts. Records from every channel see the same verdict, whether or not their channel registered the plugin.

Every registration that takes part in a solve and is live, in the plugin CEP's sense, MUST be run, subject to the plugin CEP's target platform rule.

### Consent

Configuring a channel, directly or through a relation of a configured channel, is the consent to run the plugins it registers.

A client MUST NOT run a plugin registered by a channel that takes no part in the solve.
No additional setting is needed to run a live registration. This lets users install a channel's packages without a separate plugin setup step; see [Security considerations](#security-considerations).

A client MUST offer the following controls in persistent configuration:

- Disable every registration of a channel, identified by origin, and disable a single registration, identified by origin and plugin name, as the plugin CEP requires.
- Pin a registration to a digest.
  A client MUST treat a pinned registration whose resolved digest differs as a failed plugin, reporting both digests.
  Pinning lets users review the exact code they permit to run.
- Override any plugin-provided name, or declare it absent, with `CONDA_OVERRIDE_*`.
- Clear cached verdicts and detector environments.

The plugin CEP requires clients to show what ran, its registration and digest, and any digest changes.

A client MAY require some form of opt-in before running a registered plugin.
This CEP does not prescribe an opt-in; clients that require none are conformant.

## Examples

### A channel detecting ROCm

`https://example.org/rocm-channel` builds packages against ROCm and ships the detector for it.

`linux-64/repodata.json`:

```json
{
  "info": {
    "subdir": "linux-64",
    "virtual_package_plugins": { "rocm-detect": ["__rocm"] }
  },
  "packages.conda": {
    "rocm-detect-1.0.0-h1234567_0.conda": { "name": "rocm-detect", "version": "1.0.0", "...": "..." },
    "mytool-2.0.0-h1234567_0.conda": {
      "name": "mytool", "version": "2.0.0", "depends": ["__rocm >=6.0"], "...": "..."
    }
  }
}
```

`rocm-detect` contains an executable named `rocm-detect` which prints:

```json
{ "version": 1,
  "virtual_packages": { "__rocm": { "version": "6.2.1" } },
  "cache": { "ttl_seconds": 86400, "watch_paths": ["/sys/module/amdgpu/version"] } }
```

When solving `mytool` against this channel, the client installs `rocm-detect` into a detector environment and runs it. The reported `__rocm 6.2.1` makes `mytool 2.0.0` installable.
The dependency `__rocm >=6.0` matches the single `__rocm` record contributed by the verdict, using ordinary dependency matching.
On a machine without ROCm, the plugin prints `{"version": 1, "virtual_packages": {"__rocm": null}}`. The client reports `mytool` as unsatisfiable instead of installing a package that would fail at runtime.

### One plugin, several virtual packages

```json
{ "virtual_package_plugins": { "cuda-detect": ["__cuda", "__cuda_arch"] } }
```

```json
{ "version": 1,
  "virtual_packages": {
    "__cuda": { "version": "12.4" },
    "__cuda_arch": { "version": "8.9", "build_string": "0" } } }
```

The client runs one process for both names. Keying registrations by plugin makes this grouping explicit.

### A name of one's own

`https://example.org/acme` builds against an in-house accelerator and registers a detector for it:

```json
{ "virtual_package_plugins": { "acme-detect": ["__acme_gpu"] } }
```

The distinctive name `__acme_gpu` avoids conflicts with other channels and lets `acme` ship dependent packages without coordination.
The generic name `__gpu` would be more likely to conflict with another channel's virtual packages or a name chosen by the conda community.

### Availability through a CEP 42 relation

`https://example.org/derived` declares `"channel_relations": {"base": "../rocm-channel"}` and registers nothing itself.
Resolving `derived` resolves `rocm-channel` too, so `rocm-channel`'s registration takes part in the solve and `rocm-detect` runs.
A package from `derived` with `depends: ["__rocm >=6.0"]` is matched against the verdict, exactly as a package from `rocm-channel` is.

The relation includes the registration in the solve but does not scope the verdict: both channels' packages see the same `__rocm` value.

### Two channels, one name

`https://a.example/chan-a` and `https://b.example/chan-b` register different plugin packages for `__rocm`.

When a user configures both, the first channel in the resolved channel order supplies the plugin. The other channel's `rocm-detect` is shadowed for that name and is not run.
Every package in the solve uses the winning verdict, including packages from the shadowed channel. The conflict does not cause a failure or require ordering beyond CEP 42.

This works if both channels mean ROCm. If one uses `__rocm` for its own accelerator, its packages are instead resolved against detection for different hardware; see [Naming](#naming).
Had they registered `__chan_a_rocm` and `__chan_b_rocm`, both plugins would run, both names would be available, and each channel's packages would depend on the one they meant.

### Only some subdirs

A channel serving Linux and Windows registers a plugin in `linux-64/repodata.json` only.
A client solving for `win-64` loads `win-64` and `noarch`, finds no registration, and runs nothing.
A client solving for `linux-64` loads `linux-64` and `noarch` and runs the plugin.

## Compatibility

- Clients that do not implement this CEP ignore `info.virtual_package_plugins`, as CEP 36 recommends for unrecognized `info` keys, and retain their existing behavior.
  They fail to solve packages that depend on a plugin-provided virtual package because they cannot determine whether the capability is present.
- `repodata_version` is not bumped. The additive field can be ignored by clients that do not recognize it; a version bump would make them reject otherwise usable repodata.
- Channels that register no plugins are unaffected.
- Existing packages are unaffected: no package metadata changes. A plugin is an ordinary package and requires no new fields in `index.json`.
- CEP 30 names retain their meaning. Clients remain obliged to provide them, and plugins cannot remove them.

## Security considerations

Configuring a channel allows its code to run on the user's machine before a solve completes and before any package is installed.
The plugin CEP limits the code that runs and what it can report. This section describes the consent model and its risks.

### Why configuring a channel is consent

Users who configure a channel install its packages. Installation executes link scripts under CEP 34, and each environment activation executes activation scripts.
A detector environment uses only the registering channel and its relations. Its installation executes at most the byte compilation described in the plugin CEP, and plugin execution is bounded in time and output.
It therefore runs code from an already trusted party under stricter execution limits than ordinary packages. This CEP treats that existing trust as sufficient for plugins.

The same reasoning applies to CEP 42 relations: users install packages from related channels even if they did not name those channels directly.
CEP 42 leaves separate consent for relation-loaded channels as an open question. This CEP does not require it for plugins, but a client MAY require an opt-in as [Consent](#consent) allows.

### What is new

Plugin execution differs from package installation in two respects:

- **Timing.** A plugin runs before the solve and before any transaction plan.
  Where a client performs the plugin CEP's demand scan, it runs only if the solve could reference its names. Reporting requirements let users see afterwards what ran.
  A client MAY require confirmation beforehand, as [Consent](#consent) allows.
- **Scope.** Configuring a channel for one package permits its detection code to run whenever any dependency in a solve mentions one of its names.
  Users can restrict this with per-channel or per-registration disabling and digest pinning.

### What the channel is asked to do

- Channels SHOULD treat detection plugins as security-relevant code and SHOULD sign them where the ecosystem's signing mechanisms permit (see [CEP 27](./cep-0027.md)), because they run before the user sees a transaction.
- Channels SHOULD register detectors with few dependencies and SHOULD pin those dependencies to keep the closure stable between runs.
- A channel SHOULD register a plugin only in its intended subdirs, so clients on other platforms do not have to skip it.

## Open questions

1. **Server-side validation.**
   Whether a channel serving a registration should be required to serve the corresponding package, and whether a registry should validate that at upload time.
2. **Enforcing name uniqueness.**
   This CEP assumes unique names and recommends a naming convention ([Naming](#naming)), but does not enforce uniqueness.
   Channel priority silently resolves conflicts. This works for channels detecting the same capability but not for channels using the name differently, and clients cannot distinguish those cases.
   An open question before widespread use is whether to adopt an upload-time reserved-prefix check, a registry of channel-introduced names, a client warning for shadowed registrations, or no additional mechanism.

## Future work

- A registration source in client configuration, so a user can run a plugin no channel registers or opt out of a channel's registration in favor of a build of their own.
  The plugin CEP is written so that such a source needs to define only the shape of its registrations and its consent model.

## Rejected ideas

### A separate metadata file for registrations

A `virtual-package-plugins.json` alongside `repodata.json` would avoid touching `info`.
This would add a request to every channel for a field that is usually absent, plus separate caching, validation and versioning.
Using `info` also follows CEP 42's approach for `channel_relations`, rather than isolating the field in a separate file.

### Keying registrations by virtual package rather than by plugin

`{"__cuda": "cuda-detect", "__cuda_arch": "cuda-detect"}` is easier to read and matches the lookup a client needs once it knows which names a solve references.
However, it obscures that one process provides both names. Clients would have to invert the map to avoid running the plugin twice.
Keying by plugin records that grouping directly; clients can build the inverse lookup when needed.

### Version constraints in registrations

`{"cuda-detect >=2.0": [...]}` would let a channel demand a minimum plugin version.
The key also names the executable, so it must remain a bare package name. The channel already controls which builds it serves and can remove old builds or move them to a label.
A later revision that needs per-registration options can accept an object in place of the array without breaking clients that only know the array.

### Per-channel views of a virtual package name

An earlier draft gave virtual package names per-channel meanings.
A channel's detection applied to its own packages and to packages from channels below it in the CEP 42 relation graph. This channel was called the *authority*, determined by the virtual package name and the channel serving the record.
Two unrelated channels could each register `__rocm` with a different meaning for their own packages.

This required substantial client-side changes.
A MatchSpec is matched by name and does not carry the channel of the record declaring it ([CEP 29](./cep-0029.md)). Solvers maintain one candidate set per name, so `depends: ["__rocm >=6.0"]` cannot select a channel-specific `__rocm`.
Clients would need to qualify names while building the candidate pool, rewrite dependencies to authority-derived names, and hide those internal names from diagnostics, user-written specs and lockfiles. This would add machinery to every client for an uncommon case.
It also could not resolve two unranked channels both overriding a shared channel, except by refusing to solve.

Assuming unique names retains one candidate per name without rewriting, internal names or a new channel ordering. Duplicate claims use CEP 42's existing channel priority, like packages served by multiple channels.
The tradeoff is that if two channels use a name for different capabilities, one wins for the whole solve. [Naming](#naming) addresses this by convention; [Open questions](#open-questions) discusses enforcement.

### A mandatory opt-in step

An earlier draft required explicit opt-in before any channel-registered plugin could run, but did not specify its form.
An unspecified MUST cannot be tested. Prompting on every new digest would add a step beyond configuring the channel, and CI has no user to prompt.
[Consent](#consent) instead defines required user controls, treats channel configuration as the consent boundary, and leaves further opt-in requirements to clients.

### A union over every subdir of the channel

An earlier draft merged registrations from all subdirs of a channel.
This would require fetching otherwise unused repodata and include registrations for plugins the solve cannot install.
Using only loaded subdirs avoids extra requests. Channels can register plugins in `noarch` for all platforms or in a platform subdir for that platform alone.

## Rationale

### Why the channel and not the client

Package builders know how to detect their required capabilities and already ship conda packages. Keeping detection there avoids duplicating it across clients or user configurations.
Running the same detector in every client also gives consistent verdicts, which per-client plugin interfaces cannot guarantee.

### Why names are assumed unique

A virtual package name is global in every client today, and this CEP keeps it that way.
The alternative was drafted and rejected; see [Per-channel views of a virtual package name](#per-channel-views-of-a-virtual-package-name).

### Why the plugin comes from the registering channel

Restricting the plugin package to the registering channel preserves its choice of detector.
Otherwise, a related base channel could serve a newer package with the same name and silently replace the detector with code the registering channel had not reviewed.
Dependencies use relations for resolution, but the plugin package is the code the channel explicitly registered.

### Why two CEPs

The plugin interface and channel registration have different audiences and lifetimes.
Plugin authors need the plugin CEP; channel maintainers need this one.
A future client-configuration registration source can reuse the plugin CEP unchanged while defining its own consent model. Channel consent therefore belongs here.

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

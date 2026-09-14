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

[CEP XXXX (Virtual package detection plugins)](./cep-XXXX-detection-plugins.md), below "the plugin CEP", defines what a detection plugin is and how a client runs one, but not where a client learns which plugins to run.
This CEP is the first *registration source* for it: a channel declares, in the `info` dictionary of its `repodata.json`, that a package it serves is a detection plugin for one or more virtual packages.
A client resolving that channel runs the plugin as the plugin CEP specifies, and the reported values take part in the solve exactly as a client-detected virtual package does.

The mechanism is deliberately narrow.
A plugin answers only for names its channel advertised, its dependencies come only from that channel and the channels it relates to, and its verdicts can be overridden or suppressed from the environment.
Virtual package names are assumed to be unique across channels, and a channel introducing one SHOULD build its own name into it.
Configuring a channel is what consents to running its plugins, for the reasons given in [Security considerations](#security-considerations).

## Motivation

The plugin CEP explains why detection has to be extensible.
This CEP is about who gets to extend it, and the answer is the channel, because the channel is where the knowledge already lives:

- The party that builds packages against a capability knows how to detect it, typically by depending on a vendor tool and wrapping it in a short script, and is already shipping conda packages.
- A user who adds a channel to install its packages should not need a second, undocumented step to make those packages installable.
  "Add the channel, and also install this plugin" is the workflow the conda-forge external MPI packages have been stuck with, and it does not scale past the people who already know.
- Every alternative puts the detection knowledge somewhere it has to be duplicated: in each client, or in each user's configuration.

The workarounds are worse: `CONDA_OVERRIDE_*` variables set by hand in every environment, packages that fail at runtime rather than at solve time, wrapper scripts that solve twice, or clients carrying vendor-specific detection code for hardware their authors cannot test with.

## Specification

The terms conda channel, and channel subdirectory (subdir) MUST be understood as specified by [CEP 26](./cep-0026.md).
The `repodata.json` schema is defined by [CEP 36](./cep-0036.md).
Channel relations and the resolved channel order are defined by [CEP 42](./cep-0042.md).
Registration, origin, declared names, resolution channels, live registration, closure and digest are used as the plugin CEP defines them.
Package names are compared in their normalized form, which per CEP 26 is the lowercase form.

The **loaded subdirs** of a channel, for a solve, are the subdirs the client fetched from that channel for the solve.
CEP 42 recommends that these be the subdir of the target platform and `noarch`, and this CEP assumes that; a client that loads more subdirs treats their registrations the same way.

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

A client MUST read the value of `virtual_package_plugins` as an opaque JSON value first and validate it afterwards, so that an invalid value can be reported and dropped without disturbing the parse of the rest of the document (see [Errors](#errors)).

- Each key MUST be the name of a package the declaring channel serves in one of its subdirs.
  It is the plugin name of the registration, and the plugin CEP makes it the name of the executable to run.
  It MUST therefore be a valid *package* name and MUST NOT be a virtual package name: nothing serves a virtual package, so a key carrying the `__` prefix cannot name the package a client would install.
  A key that is not a valid package name is an error, and a client encountering one MUST treat it as such rather than ignoring the entry: the key names code the client is being asked to run, and a channel that got it wrong has not said what it meant.
  A key that is a valid package name the channel does not serve is not detectable at this point; the plugin then fails at resolution, as the plugin CEP describes.
- Each value MUST be the array of virtual package names that plugin speaks for.
  One plugin MAY speak for several virtual package names.
  This is useful when an expensive query returns a set of replies: the client runs that query once and gets all of them, instead of running a set of similar plugins that each throw information away.
- Every name in every array MUST be a valid virtual package name as the plugin CEP defines it, and SHOULD contain the channel name; see [Naming](#naming).
  A name that is not valid MUST be dropped rather than carried into the solve, where it would fail later as an unusable dependency specification.
  Dropping it MUST NOT invalidate the other names of the same plugin, and MUST NOT invalidate the rest of the `repodata.json`: a client MUST parse registrations leniently and discard only what is invalid.
  A client SHOULD report what it discarded, so that a channel maintainer can find the typo.
  Plugins left with no valid name MUST be ignored.
- One array MUST hold between 1 and 16 (inclusive on both ends) names, counted before any invalid name is dropped, and the union of a plugin's arrays across the loaded subdirs MUST hold at most 16 names, counted the same way.
  An array or union outside that range is an error.
- One channel MUST NOT register more than 64 plugins, counted over the loaded subdirs.
  Together with the plugin CEP's demand scan, where a client performs it, this bounds the number of environments and processes one channel can ask for.
  A channel exceeding it is an error.
- Additional keys in `info` beyond those CEP 36 and its extensions define SHOULD, per CEP 36, be ignored by clients that do not recognize them.
  A client that does not implement this CEP therefore ignores `virtual_package_plugins` and behaves exactly as it does today.
  CEP 36 also says such keys SHOULD NOT be present; this CEP, like CEP 42, is an extension that defines one.
- If `virtual_package_plugins` is absent or empty, the channel registers no plugins.
- A `virtual_package_plugins` that is present but is not a map of registrations, including an explicit `null`, is an error.
  Absence already says "no plugins", so a channel that wrote something else meant something it failed to express, and a client MUST NOT read that as a channel with nothing to register.

#### Subdirs

Declarations are **per subdir**, like CEP 42's `channel_relations`.
A client MUST treat the registrations found in a channel's loaded subdirs as a single set.
A plugin name that appears in more than one loaded subdir is one registration whose declared names are the union of its arrays: `linux-64` registering `x` for `["__a"]` and `noarch` registering `x` for `["__a", "__b"]` is one registration of `x` for `__a` and `__b`.
Registrations in subdirs a client did not load for the solve play no part in it.
A channel MAY therefore register a plugin in only some of its subdirs, which is how a plugin relevant to one platform is kept off the others, and a channel that wants a plugin everywhere registers it in `noarch`.

Over the union, a channel MUST NOT register two plugins for the same (normalized) virtual package name, MUST NOT let one plugin register the same (normalized) virtual package name twice within one array, and MUST NOT register two names that map to the same override variable of the plugin CEP.
Within one array of one subdir a channel MUST NOT list the same (normalized) package name under two keys; a client whose JSON parser cannot see duplicate keys is not required to detect this.
Registrations that are each valid on their own MAY still collide once merged.
The registrations of one channel are a single set with nothing to order them by, so a client encountering a collision MUST treat it as an error, and MUST NOT resolve it by picking one of the colliding registrations or by silently collapsing them into one.

The declared names of a registration are the names left after invalid ones are dropped.

#### Errors

Where this section makes a registration an error, a client MUST report it to the user and MUST then ignore the channel's `virtual_package_plugins` in its entirety, in every loaded subdir, behaving as though the channel had registered no plugins at all.
A client MUST NOT reject the surrounding `repodata.json` and MUST NOT abort the solve: a channel whose registrations a client cannot make sense of is still a channel whose packages it can install, and failing the document would take every package in the subdir down with one malformed entry.

Ignoring the section as a whole rather than the offending entry is deliberate.
The errors above are all cases where the channel contradicted itself, so a client cannot tell which part of the set was meant, and acting on the remainder would be acting on a registration set whose meaning is not established.
This is distinct from an invalid *name*, which is ignored on its own and leaves the rest of the registrations standing.

#### Sharded repodata

When a channel serves sharded repodata as defined by [CEP 16](./cep-0016.md), the `virtual_package_plugins` field MAY also appear in the `info` dictionary of the shard index.
Its schema and semantics there are identical.
If a channel serves both `repodata.json` and sharded repodata, the registrations declared in both MUST be consistent.
A client reads whichever of the two it loaded and is not expected to check the other.

### Naming

A virtual package name means one thing per solve, so a name this mechanism introduces is a name taken from the whole ecosystem.
Channel priority decides which plugin answers for a contested name (see [Contested names](#contested-names)), and that is a mechanical answer to a mechanical question: it does not make two channels that meant different capabilities by one name mean the same thing, it just picks one of them and hides the other.

A channel registering a plugin for a name that no CEP standardizes SHOULD therefore make that name distinctive by building its own channel name into it: `__acme_rocm` rather than `__rocm`, for a channel named `acme`.
The resulting name MUST still satisfy the plugin CEP's rules.

The packages that depend on such a name are built by the party that registers the plugin, so no one outside that channel has to know the name, and nothing about a distinctive name makes it harder to depend on.
What it buys is that a channel never has to coordinate with a channel it has never heard of, and that a user configuring two channels never has one channel's answer about its own hardware quietly replaced by another channel's answer about different hardware that happens to share a name.

Names that CEP 30 or a later CEP standardizes are the deliberate exception: they are shared on purpose, and a channel registering a plugin for one is asking to answer for it.
Channels SHOULD NOT do so unless they intend to replace the client's own detection, which the plugin CEP allows but never lets remove the name.

### Contested names

Two **different** channels MAY register different plugins for the same name.
This is resolved the way anything served by two channels is resolved: the registration of the channel that comes first in the resolved channel order of the solve, as CEP 42 produces it from the user's channels, wins the name and shadows every other registration for it.
A channel reached by more than one relation path is one channel, as it is for CEP 42.

Shadowing settles which plugin answers for a name.
It does not settle whether the two channels meant the same capability by it, which is a question no client can answer; see [Naming](#naming).

- A shadowed name is not wanted, in the plugin CEP's sense, for the registration that lost it.
  A registration all of whose names are shadowed is therefore not live and MUST NOT be run.
  A client SHOULD report that the registration was skipped and which channel took each name, rather than silently omitting it.
- A registration shadowed for only some of its names is run if it is otherwise live, held to the full contract as the plugin CEP requires, and its verdicts for shadowed names are discarded.
- An override applies to the name, so it reaches whichever registration won the name; a registration that lost a name never sees an override for it.

Once shadowing has been applied, every name is answered by at most one registration, which is the guarantee the plugin CEP asks a registration source for.

### Resolution channels

The resolution channels of a registration are the channels CEP 42's resolution would produce if the registering channel were the only user-specified channel, using the `channel_relations` of that channel's loaded subdirs and the same depth limit the client applies elsewhere.
If that resolution fails, because of a cycle or the depth limit, the resolution channels are the registering channel alone, and the client SHOULD warn.

The plugin package itself MUST be resolved from the registering channel: the MatchSpec the plugin CEP prescribes carries the registering channel as its channel qualifier.
Its dependencies come from the resolution channels.
A client MUST NOT resolve either from any other channel the user happens to have configured.

A channel's registration therefore reaches only code that channel's own relations reach.
Resolving against the user's whole channel list would make the plugin's supply chain a property of the user's configuration rather than of the registering channel, and would let one channel's registration pull code out of an unrelated one.

The origin of a registration is the registering channel's base URL as CEP 26 defines it.

### Which registrations take part in a solve

A registration takes part in a solve when its channel does: because the user configured the channel, or because a CEP 42 relation of a configured channel brought it in.
Channel relations bring a registration into a solve; they do not scope a verdict once it is there.
A record served by any channel sees the same verdict, whether or not the channel that served it registered the plugin.

Every registration that takes part in a solve and is live, in the plugin CEP's sense, MUST be run, subject to the plugin CEP's target platform rule.

### Consent

Configuring a channel, directly or through a relation of a configured channel, is the consent to run the plugins it registers.

A client MUST NOT run a plugin registered by a channel that takes no part in the solve.
Nothing beyond configuring the channel is needed for a live registration to run, so that a user who adds a channel gets its packages installable without discovering a second setting.
The rationale is in [Security considerations](#security-considerations).

What a user can always do, and a client MUST offer in persistent configuration:

- Disable every registration of a channel, identified by origin, and disable a single registration, identified by origin and plugin name, as the plugin CEP requires.
- Pin a registration to a digest.
  A client MUST treat a pinned registration whose resolved digest differs as a failed plugin, reporting both digests.
  This is the exact-trust mode for users who want to review what runs.
- Override any plugin-provided name, or declare it absent, with `CONDA_OVERRIDE_*`.
- Clear cached verdicts and detector environments.

What a user can always see: what ran, from which registration, with which digest, and when a digest changed, as the plugin CEP requires of every client.

A client MAY require some form of opt-in before running a registered plugin.
This CEP does not specify one, and a client that requires none is conformant.

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

Solving `mytool` against this channel: the client installs `rocm-detect` into a detector environment, runs it, obtains `__rocm 6.2.1`, and `mytool 2.0.0` becomes installable.
`mytool`'s `__rocm >=6.0` is matched against `6.2.1` like any other dependency, against the one `__rocm` record the verdict contributed.
On a machine without ROCm the same plugin prints `{"version": 1, "virtual_packages": {"__rocm": null}}` and `mytool` is correctly reported as unsatisfiable rather than installing and failing at runtime.

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

One process answers for both, which is why registrations are keyed by plugin rather than by virtual package: the client runs it once.

### A name of one's own

`https://example.org/acme` builds against an in-house accelerator and registers a detector for it:

```json
{ "virtual_package_plugins": { "acme-detect": ["__acme_gpu"] } }
```

Nothing else in the ecosystem is likely to use `__acme_gpu`, so no other channel can contradict it, and `acme` can ship packages depending on it without coordinating with anyone.
Had `acme` registered `__gpu` instead, it would have taken a name any other channel (or the conda community) might reasonably want for its own virtual packages.

### Availability through a CEP 42 relation

`https://example.org/derived` declares `"channel_relations": {"base": "../rocm-channel"}` and registers nothing itself.
Resolving `derived` resolves `rocm-channel` too, so `rocm-channel`'s registration takes part in the solve and `rocm-detect` runs.
A package from `derived` with `depends: ["__rocm >=6.0"]` is matched against the verdict, exactly as a package from `rocm-channel` is.

The relation is what brought the registration into the solve; it does not scope the verdict.
`__rocm` has one value here, and both channels' packages see it.

### Two channels, one name

`https://a.example/chan-a` and `https://b.example/chan-b` both register a plugin, different packages, for `__rocm`.

A user configuring both gets one `__rocm`: whichever of the two channels comes first in the resolved channel order supplies the plugin, and the other channel's `rocm-detect` is shadowed for that name and not run.
Every package in the solve is matched against the winning verdict, including packages from the channel that lost.
Nothing fails, and nothing has to be ranked that CEP 42 does not already rank.

That is the right outcome when both channels meant ROCm, and the wrong one when they did not: a channel that meant its own accelerator by `__rocm` has just had its detection replaced by a detector for someone else's hardware, and the packages depending on it will be resolved against an answer to a different question.
This is what [Naming](#naming) is for.
Had they registered `__chan_a_rocm` and `__chan_b_rocm`, both plugins would run, both names would be available, and each channel's packages would depend on the one they meant.

### Only some subdirs

A channel serving Linux and Windows registers a plugin in `linux-64/repodata.json` only.
A client solving for `win-64` loads `win-64` and `noarch`, finds no registration, and runs nothing.
A client solving for `linux-64` loads `linux-64` and `noarch` and runs the plugin.

## Compatibility

- **Clients that do not implement this CEP** ignore `info.virtual_package_plugins`, as CEP 36 recommends for unrecognized `info` keys, and behave exactly as before.
  They will fail to solve packages that depend on a plugin-provided virtual package, correctly, since they cannot determine whether the capability is present.
- **`repodata_version` is not bumped.** The field is additive, and clients that do not know it ignore it.
  Bumping the version would force every client to reject repodata it can otherwise use.
- **Existing channels are unaffected.** A channel that registers nothing behaves identically.
- **Existing packages are unaffected.** No package's metadata changes; a plugin is an ordinary package and requires no new fields in `index.json`.
- **The CEP 30 names keep their meaning.** A client's obligation to provide them is unchanged, and a plugin cannot remove one.

## Security considerations

**This CEP describes a mechanism by which configuring a channel causes code from that channel to be executed on the user's machine, before any solve completes and before any package is installed.**
The plugin CEP bounds what that code is and what it can report; this section is about why configuring the channel is taken as consent, and what that costs.

### Why configuring a channel is consent

A user who configures a channel installs its packages.
Installing a package executes its link scripts under CEP 34, and every activation of the resulting environment executes its activation scripts.
A detector environment is resolved from that same channel and its relations and nothing else, installing it executes at most the byte compilation the plugin CEP describes, and running it is bounded in time and output.
Running a registered plugin therefore runs code from a party the user already runs code from, under a stricter bound than any package enjoys.
Asking for consent a second time would be asking the user to trust a subset of what they already trust.

Consent extends through CEP 42 relations for the same reason: a related channel is a channel the user installs packages from, whether or not they typed its name.
CEP 42 lists whether relation-loaded channels need separate consent as an open question; this CEP takes the position that, for plugins, they do not, and a client that disagrees MAY require an opt-in as [Consent](#consent) allows.

### What is new

Two things are new relative to installing a package, and they are the honest cost of this CEP:

- **Timing.** A plugin runs before the solve, so before a client that shows a transaction plan has shown it.
  Where a client performs the plugin CEP's demand scan, a plugin runs only when the solve could reference its names, and the reporting requirements mean a user can always see afterwards what ran.
  A client that wants a confirmation before that point MAY require one, as [Consent](#consent) allows.
- **Scope.** A user who configured a channel for one package now runs that channel's detection whenever any dependency in a solve mentions one of its names.
  Per-channel and per-registration disabling, and digest pinning, are the tools for a user who wants less than that.

### What the channel is asked to do

- Channels SHOULD treat a detection plugin as security-relevant code and SHOULD sign it where the ecosystem's signing mechanisms permit (see [CEP 27](./cep-0027.md)), because it is the one package that runs before the user has seen a transaction.
- Channels SHOULD register detectors with few dependencies and SHOULD pin the ones they have, so that the closure a user sees today is the closure that runs tomorrow.
- A channel SHOULD register a plugin only in the subdirs it is meant for, so that clients on other platforms do not have to skip it.

## Open questions

1. **Server-side validation.**
   Whether a channel serving a registration should be required to serve the corresponding package, and whether a registry should validate that at upload time.
2. **Enforcing name uniqueness.**
   This CEP assumes names are unique and recommends a convention for keeping them so ([Naming](#naming)), but nothing enforces it.
   Where two channels claim one name, channel priority silently picks one, which is right when they meant the same capability and wrong when they did not, and a client cannot tell the two cases apart.
   Whether the ecosystem wants a reserved-prefix rule a registry can check at upload time, a registry of the names channels have introduced, a client warning when a registration is shadowed, or nothing at all, is worth settling before this is widely used.

## Future work

- A registration source in client configuration, so a user can run a plugin no channel registers or opt out of a channel's registration in favor of a build of their own.
  The plugin CEP is written so that such a source needs to define only the shape of its registrations and its consent model.

## Rejected ideas

### A separate metadata file for registrations

A `virtual-package-plugins.json` alongside `repodata.json` would avoid touching `info`.
Rejected: it is an extra request on every channel for a field that is almost always absent, and it would need its own caching, validation and versioning.
CEP 42 made the same call for `channel_relations`, and consistency with it is worth more than the isolation.

### Keying registrations by virtual package rather than by plugin

`{"__cuda": "cuda-detect", "__cuda_arch": "cuda-detect"}` reads more naturally, and it is the direction a client looks things up in once it knows which names a solve needs.
Rejected: it hides that one process answers for both names, and a client would have to invert the map to avoid running the plugin twice.
A client that wants the inverse builds it; the channel writes down the fact that is true, which is that one plugin answers for a set of names.

### Version constraints in registrations

`{"cuda-detect >=2.0": [...]}` would let a channel demand a minimum plugin version.
Rejected: the key is the executable's name and has to stay a bare package name, and the channel already controls which builds it serves.
A channel that wants users off an old build stops serving it, or moves it to a label.
A later revision that needs per-registration options can accept an object in place of the array without breaking clients that only know the array.

### Per-channel views of a virtual package name

An earlier draft of this CEP let a name mean different things to different channels.
A channel answered for a name over the packages it serves and over the packages of channels below it in the CEP 42 relation graph; the channel that answered was called the *authority*, and it was a function of the name and the channel that served the record.
Two unrelated channels could then each register `__rocm`, and each be right for its own packages.

Rejected as too expensive for what it buys.
A MatchSpec is matched by name and carries nothing about the channel of the record that declared it ([CEP 29](./cep-0029.md)), and solvers keep one candidate set per name, so `depends: ["__rocm >=6.0"]` cannot find "the right `__rocm`" on its own.
Implementing views therefore meant qualifying names internally while the candidate pool was built, rewriting every dependency on such a name to a name derived from its authority, and then keeping those internal names out of diagnostics, out of user-written specs and out of lockfiles: a whole layer of machinery in every client, in service of a case that mostly does not arise.
It also left a case that no channel had decided, two channels overriding one shared channel, neither ranking above the other, for which it had no answer better than refusing to solve.

Assuming names are unique removes all of it.
One candidate per name, no rewriting, no internal names, and no new ordering relation over channels: a name claimed twice is settled by the channel priority CEP 42 already produces, the way a package served twice is.
The cost is that two channels choosing one name for two different capabilities is no longer harmless, one of them wins for everyone, which [Naming](#naming) addresses by convention and [Open questions](#open-questions) leaves open to address by enforcement.

### A mandatory opt-in step

An earlier draft required every client to obtain an explicit opt-in before running any channel-registered plugin, without saying what form it takes.
Rejected: a MUST whose content is unspecified is not testable, a prompt on every new digest is the opposite of "add the channel and it works", and there is nobody to prompt in CI.
[Consent](#consent) instead fixes what a user can always do, makes the consent boundary the one users already reason about, and leaves any further opt-in to clients.

### A union over every subdir of the channel

An earlier draft merged the registrations of all subdirs of a channel.
Rejected: a client would have to fetch repodata it has no other use for, and a registration in a subdir the solve does not touch describes a plugin the solve cannot install.
Restricting the union to the loaded subdirs makes "register in `noarch` for everywhere, in a platform subdir for that platform" the natural spelling, and keeps registration lookup free.

## Rationale

### Why the channel and not the client

The party that builds packages against a capability is the party that knows how to detect it, and is already shipping conda packages.
Every alternative puts that knowledge somewhere it has to be duplicated: in each client, or in each user's configuration.

Detection written once by that party, run identically by every client, is also what makes verdicts agree across clients, which a per-client plugin interface cannot promise.

### Why names are assumed unique

A virtual package name is global in every client today, and this CEP keeps it that way.
The alternative was drafted and rejected; see [Per-channel views of a virtual package name](#per-channel-views-of-a-virtual-package-name).

### Why the plugin comes from the registering channel

A registration says "run my package".
If the plugin itself could resolve from a related channel, a base channel serving a newer package of the same name would silently replace the registrant's detector with its own, and the registrant would have registered code it never saw.
Dependencies may come from relations because that is what relations are for; the plugin may not, because that is what the registration is for.

### Why two CEPs

The plugin interface and the channel registration have different audiences and different lifetimes.
A plugin author needs the plugin CEP and nothing else.
A channel maintainer needs this one.
A future registration source in client configuration needs the plugin CEP unchanged and a consent model of its own, which is why the consent model lives here rather than there.

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

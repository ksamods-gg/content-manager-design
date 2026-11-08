---
rfc: "0000"
title: Loader stand-ins
status: Draft
authors: ["@SafeShows"]
created: 2026-11-08
discussion:
supersedes: []
superseded-by: []
---

# RFC 0000: Loader stand-ins

## Summary

A `mod-loader` listing may add `[[provides.stand-in]]` entries, each naming another loader and a mod that runs on it, to say that this loader alone does what that loader and that mod do together.
A manager may then install a mod that is listed for that loader and requires that mod into an instance that runs the stand-in, without installing either of them, and checks the mod's bounds on the required mod against the stand-in's version.
The stand-in and the mod it replaces must share their release authority, so only their author can claim it and their versions always come from the same releases.
Mod listings do not change, a listing keeps exactly one `[loader]`, and `spec_version` and `snapshot_version` stay at `1`.

## Motivation

Oxidizer runs Kitten Space Agency mods written in Rust or as Rhai scripts, and it is listed twice, from the same repository and the same releases.
`Oxidizer.StarMap` is a `mod` that StarMap loads like any C# mod, and that then runs the Oxidizer mods next to StarMap's C# mods.
`Oxidizer` is a `mod-loader` that starts the game on its own, without StarMap, and runs the same Oxidizer mods.
Both read the same folders: `mods/<id>/mod.toml` with an `[oxidizer]` section (verified: both are built from one Rust crate, and the standalone launcher and the StarMap mod call the same loading code).

As agreed with the index stewards, an Oxidizer mod is listed with `[loader] id = "StarMap"` and a required dependency on `Oxidizer.StarMap`, because a listing has one loader and there is no choice of loaders across `any_of`.
That works for StarMap players, but a manager cannot install the same mod for a player who runs standalone Oxidizer, although the mod runs there unchanged: the mod names StarMap as its loader, and the instance does not run StarMap.
The first such listing is `flight_panel`.

If we do nothing, Oxidizer mod authors choose between two listings of one mod and leaving standalone players to install by hand, and two listings are worse than they look: an id is the folder name ([RFC 0031](0031-content-metadata-format.md)), so `flight_panel` and `flight_panel-standalone` would be two folders holding one mod.

The general shape is a loader that can run, by itself, mods made for another loader plus one specific mod on it.
Oxidizer is the case at hand; any loader that later ships both as a StarMap mod and as its own launcher is the same case.

## Guide-level explanation

### If you ship a loader both ways

Your standalone listing says which loader and mod it stands in for:

```toml
# listings/Oxidizer.toml
type = "mod-loader"

[releases]
github = "SafeShows/Oxidizer"

[provides]
launch = "oxidizer.exe"
content-dir = "mods"

[[provides.stand-in]]
loader = "StarMap"
mod = "Oxidizer.StarMap"
```

Your mod listing `Oxidizer.StarMap` names the same `[releases]` authority, `SafeShows/Oxidizer`, which is what lets the index accept the claim.

### If you write mods for it

Nothing changes.
List your mod for StarMap, with the required dependency, as today:

```toml
# listings/flight_panel.toml
[loader]
id = "StarMap"
min = "0.4.6"

[[dependencies]]
id = "Oxidizer.StarMap"
kind = "required"
min = "0.4.0"
```

A player who runs StarMap gets StarMap, `Oxidizer.StarMap` 0.4.0 or later and `flight_panel`, as today.
A player whose instance runs standalone Oxidizer gets `flight_panel` only, and the manager checks that the installed Oxidizer is 0.4.0 or later.

### If your mod also needs C# mods

A dependency that is itself a StarMap mod, but not one the stand-in replaces, still needs StarMap.
The manager cannot install it under standalone Oxidizer, and says so the way it says so today for a mod whose loader the instance does not run.

## Reference-level explanation

### The entry

`[[provides.stand-in]]` is an optional array of tables inside `[provides]` of RFC 0035, so it is permitted only where `type = "mod-loader"`.

| Key | Type | Required | Meaning |
|---|---|---|---|
| `loader` | string | yes | The id of the loader this listing stands in for. Must reference content of type `mod-loader`, and not this listing. |
| `mod` | string | yes | The id of the mod this listing does the work of. Must reference content of type `mod` whose `[loader]` names `loader`. |

The checks of the authored repository reject an entry when:

- either reference does not resolve to the type above,
- two entries of one listing name the same `loader` and `mod`,
- `mod` does not name the same release authority as this listing: the same host and the same repository (or SpaceDock id) under `[releases]`,
- the stand-in's `[provides].content-dir` differs from the `content-dir` of `loader`, because a manager would install the mod where the stand-in does not read it.

The shared authority is the ownership proof: [RFC 0033](0033-content-index.md) binds a listing to its authority, so only the author of both releases can claim that one replaces the other.
It is also what makes the versions comparable: the watcher stamps both listings from the same releases, so the stand-in's version and the mod's version are the same number for the same code.

### Which mods it covers

A release of a mod qualifies for an entry when its stamped `loader.id` is the entry's `loader` and its dependencies hold a `required` entry on the entry's `mod`, authored or derived.
A `recommends` entry does not qualify: the mod runs without the replaced mod, so on the stand-in it needs the replaced loader for something else.

### What a manager does

In an instance that runs a stand-in, a manager may offer a qualifying release for install as follows:

1. It does not install the entry's `loader` or `mod`.
2. It satisfies the required dependency on `mod` with the installed stand-in: the dependency's `min` and `max` apply to the stand-in's installed version, and a stand-in outside them is an unsatisfied dependency, reported as one.
3. It ignores the `[loader]` bounds of the release, which constrain a loader that is not involved.
4. It resolves every other dependency as today, and a dependency that needs a loader the instance does not run stays unresolvable, as today.
5. It installs the mod under the stand-in's `content-dir`, by the mod's `[install]` descriptor of RFC 0035.

A pack pins mods, not loaders ([RFC 0031](0031-content-metadata-format.md)), so a pack whose members all qualify can be installed under the stand-in by the same rules, and a pack member that pins the replaced `mod` is satisfied by the stand-in when the pinned version equals the stand-in's version.

A manager that does not know the entry ignores it and behaves as today.

### Files

The release files do not change.
The snapshot copies `provides` of the listing verbatim, as it does today, so the entries reach clients without a snapshot change.
`spec_version` and `snapshot_version` stay at `1`.

## Drawbacks

- A manager learns a second way to satisfy a dependency, by a loader instead of a mod, which every resolver has to implement before a player benefits.
- The check that both listings share an authority constrains how a loader author can split their repositories: a stand-in and its mod must come from one repository.
- Nothing in the index proves that the stand-in really runs every qualifying mod; the author asserts it, as an author asserts compatibility today.

## Alternatives

- **`any_of` over loaders on the mod.** Every mod author would repeat the choice, and the index stewards prefer one loader per listing.
- **Two listings per mod**, one per loader. Two ids are two folder names, so a player who installs both runs the mod twice.
- **The stand-in provides only the mod**, without naming the loader. A manager would still find `[loader] id = "StarMap"` on the mod and refuse, so the loader has to be part of the claim.
- **A loader stand-in without a mod** (standalone Oxidizer stands in for StarMap). False: standalone Oxidizer does not run C# mods, so it would claim every StarMap mod.

## Unresolved questions

- **An instance that runs both.** Whether a manager should prefer the stand-in or the replaced loader when a player could choose either, and whether it should offer to switch an instance between them, is left to managers.
- **A stand-in for a loader with its own instance handover** ([RFC 0049](0049-instance-handover.md)) works today because both are started by their own `[provides.instance]`, but this RFC does not check that a stand-in passes an instance on the same way.

## Future possibilities

- A manager could show a player which installed mods a stand-in would also run, and offer the lighter setup.
- The same entry could describe a loader that a later loader version absorbs, so a rename of a loader does not strand the mods listed for the old one.

---
rfc: "0000"
title: Listing metadata bag
status: Draft
authors: ["@SafeShows"]
created: 2026-09-06
discussion:
supersedes: []
superseded-by: []
---

# RFC 0000: Listing metadata bag

## Summary

An optional `[metadata]` table on every authored file, holding namespaced key-value data that no resolver, installer, or compatibility check may read.

It is an extension point, not a field.
A client or an index puts data under a namespace it owns, every other consumer ignores it, and a key that several consumers converge on becomes a first-class field by RFC.

## Motivation

RFC 0031 says clients ignore fields they do not know, which is what makes adding an optional field cheap.
It does not say where a consumer may put data of its own, so today there are two answers and both are bad.

**Abuse `[links]`.**
It already accepts arbitrary keys, so anything shaped like a URL can hide there.
That is where a Borea-specific accent colour or a ksamods.gg featured video will end up, and every client that renders `[links]` as a link list then renders them as broken links.

**Keep it out of the format.**
ksamods.gg serves its own answer from its own API, Borea keeps a side table, and the same fact exists twice with no way to tell which copy is current.
This is the position the whole of RFC 0031 was written to get out of, reintroduced one layer down.

Doing nothing does not stop consumers carrying extra data.
It decides only whether that data sits in a named place with rules, or in `[links]` without any.

## Guide-level explanation

You add a table named after whoever consumes the data:

```toml
id = "AdvancedFlightComputer"
type = "mod"
# ... the rest of the authored file

[metadata.borea]
accent = "#ff8800"

[metadata.ksamods-gg]
featured-video = "https://www.youtube.com/watch?v=..."
banner = "https://example.com/afc-banner.png"
```

Borea reads `metadata.borea` and ignores the rest.
ksamods.gg reads `metadata.ksamods-gg` and ignores the rest.
A third client reads neither and loses nothing, because nothing here changes what gets installed.

That last part is the rule that matters: **nothing under `[metadata]` may change what any client does.**
If a fact decides whether a mod installs, resolves, or is compatible, it is a first-class field and needs an RFC.
The bag is for things that only change how a listing looks.

## Reference-level explanation

### Where it lives

In the shared authored core of [RFC 0031](0031-content-metadata-format.md), so it applies to `mod`, `modpack`, `mod-loader`, and every type added later with no per-type work.
Optional everywhere; absent means nothing.

### Shape

`[metadata]` is a table of tables.
Each direct child is a **namespace**, and its contents are whatever TOML that namespace's owner wants.

| Rule | Reason |
|---|---|
| Every direct child of `[metadata]` is a table. A scalar or array directly under `[metadata]` makes the file invalid. | Un-namespaced keys are how two consumers end up fighting over `color`. Forcing the namespace makes a collision impossible rather than unlikely. |
| Namespace names follow RFC 0031's id rules, compared case-insensitively. | Reusing the id charset means no second naming scheme to specify, and it is already known to be safe on every platform. |
| A namespace need not be a listed id, and claiming one grants nothing. | The bag names consumers, not content, and `borea` is not a mod. |
| The serialized `[metadata]` table is at most 4 KiB per listing. | Every byte lands in the snapshot RFC 0033 ships to every client, so an unbounded bag is a cost paid mostly by people who never read it. |

### What it may not do

1. **No consumer may read a namespace it does not own.** Reading someone else's namespace makes their private key your public contract, and they will change it.
2. **Nothing under `[metadata]` may affect resolution, dependencies, compatibility, install, load order, or versioning.** A validator cannot enforce this, so it is stated as the boundary the promotion path exists to keep: a fact that needs to change behaviour is a first-class field, and an RFC is the cost of that.
3. **It is not stamped into release files.** RFC 0031's `listing` block freezes the display facts a release shipped with; this is live presentation data owned by consumers rather than by the release, and a frozen copy would only go stale.
4. **It is authored text like any other.** RFC 0033's moderation, delisting, and takedown path applies unchanged.

### Promotion

A key that several consumers independently carry is evidence, not a decision.
When one shows up in more than one namespace, the answer is an RFC adding a real field with a real meaning, after which the namespaced copies are redundant and get dropped at the author's leisure.
The bag exists so that evidence can accumulate without a format change every time somebody wants to try something.

### Errors

| Condition | Result |
|---|---|
| A scalar or array directly under `[metadata]` | File invalid. |
| A namespace name that is not a valid id | File invalid. |
| Two namespaces differing only by case | File invalid, same as ids. |
| Serialized `[metadata]` over 4 KiB | File invalid. |

`spec_version` stays at `1`: this is an optional field, and RFC 0031's evolution rule already says adding one is not a break.

## Drawbacks

- **It is a place to put things instead of deciding what they are.** Every field that should have been argued through an RFC can now be shipped quietly under a namespace, and the promotion path is a convention with nothing enforcing it.
- **Rule 2 is unenforceable.** Nothing stops a client resolving on `metadata.itself.requires`, and nobody outside that client would know.
- **It makes the format partly opaque.** A reader of a listing can no longer tell what all of it means, which is a real loss for a format whose selling point is that one file says everything.
- **4 KiB is a made-up number.** Small enough to be safe, large enough for anything presentational, and the first consumer that hits it will have a good argument for why the limit is wrong.
- **Namespaces are unowned.** Anyone can write `[metadata.borea]` into their own listing, so Borea has to treat its own namespace as untrusted input from the author, which it would anyway.

## Alternatives

**Do nothing, and let `[links]` carry it.**
Free, and already how it would happen.
Rejected because `[links]` has a stated meaning that clients render, so anything non-link parked there is a bug in every other client, and anything that is not a URL does not fit at all.

**A first-class field per need.**
Correct, and what the promotion path leads to.
Rejected as the only mechanism because it puts an RFC in front of trying anything, which is how you get the private side tables this RFC exists to prevent.

**One flat un-namespaced table.**
Shorter to write and to specify.
Rejected because the first collision is unfixable: two consumers using `color` for different things cannot both be right, and there is no authority to arbitrate.

**Let each index define its own extension block.**
The status quo, portable to nobody.
It also puts index-specific data in the one place RFC 0031 established as index-independent.

## Unresolved questions

1. **Is 4 KiB the right cap, and is it per listing or per namespace?** Per listing is simpler and makes namespaces compete for it, which may be the wrong incentive.
2. **Should namespaces be registered anywhere?** Unowned is simplest and matches the ecosystem's size; a squatted namespace has no remedy today.
3. **Does the snapshot carry `[metadata]` for every listing, or may a client fetch it separately?** Small enough not to matter now, but it is the first field whose readers are a strict subset of clients.

## Future possibilities

- **Promotion of whatever converges**, which is the point.
- **A registry of known namespaces**, if squatting ever becomes real rather than theoretical.
- **The same bag on a pack's member entries**, if a curator ever needs to annotate one pinned mod rather than the pack.

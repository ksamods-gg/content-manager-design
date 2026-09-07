---
rfc: "0051"
title: Listing extensions table
status: Proposed
authors: ["@SafeShows"]
created: 2026-09-06
discussion: https://github.com/KSAModding/content-manager-design/pull/51
supersedes: []
superseded-by: []
---

# RFC 0051: Listing extensions table

## Summary

An optional `[extensions]` table on every authored file, holding namespaced key-value data that no resolver, installer, or compatibility check may read.

It is an extension point, not a field.
A client or an index puts data under a namespace it owns, every other consumer ignores it, and a key that several consumers converge on becomes a first-class field by RFC.

## Motivation

**There is no extension point at all today, and that is a stricter starting position than it sounds.**

RFC 0031 says clients ignore fields they do not know, which is what makes adding an optional field cheap.
The authored schema the index validates against, `schemas/authored.schema.json` in `content-index`, is not so relaxed: it sets `additionalProperties: false` at the top level, so a listing carrying any table the schema does not name is rejected at publish time.
`[links]` is no escape hatch either, because every value under it is constrained to a URL pattern, so a colour or an identifier does not get in.

The format is closed, and closed is the right default.
What follows from it is that a consumer with data of its own has exactly one option: keep it out of the format.
ksamods.gg serves its own answer from its own API, Borea keeps a side table, and the same fact exists twice with no way to say which copy is current.
That is the position the whole of RFC 0031 was written to get out of, reintroduced one layer down.

Doing nothing does not stop consumers carrying extra data.
It decides only whether that data lives somewhere with rules, or in private stores nobody else can see.

## Guide-level explanation

You add a table named after whoever consumes the data:

```toml
id = "AdvancedFlightComputer"
type = "mod"
# ... the rest of the authored file

[extensions.borea]
accent = "#ff8800"

[extensions.ksamods-gg]
legacy-id = 4253
```

Borea reads `extensions.borea` and ignores the rest.
ksamods.gg reads `extensions.ksamods-gg` and ignores the rest.
A third client reads neither and loses nothing.

Both examples are deliberately narrow.
`accent` is one client's theming choice, and `legacy-id` is the row this listing came from in an index's old database, useful to that index and meaningless to anyone else.
That is the shape of an extension: **data genuinely private to one consumer.**

A per-listing icon or banner is the opposite of that shape.
Every client wants images, which makes them a first-class field, and [#45](https://github.com/KSAModding/content-manager-design/discussions/45) is where they are being decided as the gap they are in `spec_version = 1`.
They do not belong in this table, and a namespace carrying one is an argument for #45 rather than a use of this RFC.

The line an extension must not cross: it may change how a listing is **presented**, never what is **resolved, downloaded, or installed**.
An accent colour changes what Borea draws, and that is the entire point of it.
A key that decides which version resolves, whether a mod is compatible, or where it unpacks is a first-class field and needs an RFC.

## Reference-level explanation

### Where it lives

In the shared authored core of [RFC 0031](0031-content-metadata-format.md), so it applies to `mod`, `modpack`, `mod-loader`, and every type added later with no per-type work.
Optional everywhere; absent means nothing.

It is named `[extensions]` rather than `[metadata]` because the entire file is metadata, so the broader word says nothing about what is inside.

### Shape

`[extensions]` is a table of tables.
Each direct child is a **namespace**, and its contents are whatever TOML that namespace's owner wants.

| Rule | Reason |
|---|---|
| Every direct child of `[extensions]` is a table. A scalar or array directly under `[extensions]` makes the file invalid. | Un-namespaced keys are how two consumers end up fighting over `accent`. Forcing the namespace makes a collision impossible rather than unlikely. |
| Namespace names follow RFC 0031's id rules, compared case-insensitively. | Reusing the id charset means no second naming scheme to specify, and it is already known to be safe on every platform. |
| A namespace need not be a listed id, and claiming one grants nothing. | The table names consumers, not content, and `borea` is not a mod. |
| The serialized `[extensions]` table is at most 4 KiB per listing. | Every byte lands in the snapshot RFC 0033 ships to every client, so an unbounded table is a cost paid mostly by people who never read it. |

### What it may not do

1. **No consumer may read a namespace it does not own.** Reading someone else's namespace makes their private key your public contract, and they will change it.
2. **Nothing under `[extensions]` may affect resolution, dependencies, compatibility, install, load order, or versioning.** Presentation is exactly what it is for; behaviour is what it is not. A validator cannot tell the two apart, so this is stated as the boundary the promotion path exists to keep, not as something the index can enforce.
3. **It is not stamped into release files.** RFC 0031's `listing` block freezes the display facts a release shipped with; this is live data owned by consumers rather than by the release, and a frozen copy would only go stale.
4. **It is authored text like any other.** RFC 0033's moderation, delisting, and takedown path applies unchanged.

### Promotion

A key that several consumers independently carry is evidence, not a decision.
When one shows up in more than one namespace, the answer is an RFC adding a real field with a real meaning, after which the namespaced copies are redundant and get dropped at the author's leisure.
Icons and banners in #45 are that path already running ahead of this RFC: wanted by everyone, therefore first-class, therefore not extensions.

### Companion change

This RFC does not take effect on its own.
`schemas/authored.schema.json` in `content-index` rejects unknown top-level tables, so the schema has to permit `extensions` and validate its shape, as a pull request against `KSAModding/content-index` in the same shape as the one RFC 0048 carries as [content-index#34](https://github.com/KSAModding/content-index/pull/34).
Accepting this RFC without that change lands a field the index refuses at publish time.

### Errors

| Condition | Result |
|---|---|
| A scalar or array directly under `[extensions]` | File invalid. |
| A namespace name that is not a valid id | File invalid. |
| Two namespaces differing only by case | File invalid, same as ids. |
| Serialized `[extensions]` over 4 KiB | File invalid. |

`spec_version` stays at `1`: this is an optional field, and RFC 0031's evolution rule already says adding one is not a break.

## Drawbacks

- **It is a place to put things instead of deciding what they are.** Every field that should have been argued through an RFC can now be shipped quietly under a namespace, and the promotion path is a convention with nothing enforcing it.
- **The presentation-versus-behaviour line is unenforceable.** Nothing stops a client resolving on `extensions.itself.requires`, and nobody outside that client would know.
- **It makes the format partly opaque.** A reader of a listing can no longer tell what all of it means, which is a real loss for a format whose selling point is that one file says everything.
- **It reopens `additionalProperties: false`.** The index's schema is closed today and catches typos and junk for free; one permissive subtree is a small hole, but it is the first one.
- **4 KiB is a made-up number.** Small enough to be safe, large enough for anything presentational, and the first consumer that hits it will have a good argument for why the limit is wrong.
- **Namespaces are unowned.** Anyone can write `[extensions.borea]` into their own listing, so Borea has to treat its own namespace as untrusted input from the author, which it would anyway.

## Alternatives

**Do nothing.**
Free, and the index enforces it today: the schema is closed, so consumers keep private side tables and the same fact lives in two places.
Rejected because that is the duplication RFC 0031 exists to remove, not an absence of the problem.

**Relax `[links]` to carry non-URL values.**
Tempting because the table already accepts arbitrary keys.
Rejected because the URL constraint is the only reason a client can render `[links]` as links without checking each one, and because nothing shaped like a colour or a row id belongs under that name anyway.

**A first-class field per need.**
Correct, and what the promotion path leads to.
Rejected as the only mechanism because it puts an RFC in front of trying anything, which is how you get the private side tables this RFC exists to prevent.

**One flat un-namespaced table.**
Shorter to write and to specify.
Rejected because the first collision is unfixable: two consumers using `accent` for different things cannot both be right, and there is no authority to arbitrate.

## Unresolved questions

1. **Is 4 KiB the right cap, and is it per listing or per namespace?** Per listing is simpler and makes namespaces compete for it, which may be the wrong incentive.
2. **Should namespaces be registered anywhere?** Unowned is simplest and matches the ecosystem's size; a squatted namespace has no remedy today.
3. **How far does the schema open?** Permitting `extensions` with typed namespaces still validates more than permitting a free subtree, and the companion pull request has to pick one.
4. **Does the snapshot carry `[extensions]` for every listing, or may a client fetch it separately?** Small enough not to matter now, but it is the first field whose readers are a strict subset of clients.

## Future possibilities

- **Promotion of whatever converges**, which is the point.
- **A registry of known namespaces**, if squatting ever becomes real rather than theoretical.
- **The same table on a pack's member entries**, if a curator ever needs to annotate one pinned mod rather than the pack.

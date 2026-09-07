---
rfc: "0051"
title: Listing extensions table
status: Postponed
authors: ["@SafeShows"]
created: 2026-09-06
discussion: https://github.com/KSAModding/content-manager-design/pull/51
supersedes: []
superseded-by: []
---

# RFC 0051: Listing extensions table

## Summary

An optional `[extensions]` table on every authored file, holding namespaced key-value data that no resolver, installer, or compatibility check may read.

**Postponed.**
Review established that the derived-view model accepted in [#37](https://github.com/KSAModding/content-manager-design/discussions/37) already covers every case this RFC could name, and that nobody could produce the one kind of fact an extensions table would be needed for.
The design is recorded below, with the corrections review produced, so that a future proposal starts from the corrected version rather than this one's first draft.
The bar for reopening is stated at the end.

## Motivation

**There is no extension point today, and that is a stricter starting position than it sounds.**

RFC 0031 says clients ignore fields they do not know, which is what makes adding an optional field cheap.
The authored schema the index validates against, `schemas/authored.schema.json` in `content-index`, is not so relaxed: it sets `additionalProperties: false` at the top level, so a listing carrying any table the schema does not name is rejected at publish time.
`[links]` is no escape hatch either, because every value under it is constrained to a URL pattern, so a colour or an identifier does not get in.

The original argument from there was that a consumer with data of its own therefore has to keep it outside the format, which duplicates the fact and leaves nothing to say which copy is current.

**That argument was wrong, and the reason it was wrong is why this RFC is postponed.**

## Why this is postponed

### The duplication it claimed to remove is the duplication it would create

Service-local data already has an accepted home: the derived-view model in [#37](https://github.com/KSAModding/content-manager-design/discussions/37), where a service keeps its own data in its own store, keyed by the canonical listing id.
There is one copy, in the store whose owner maintains it, and the listing id is the join.

Both of this RFC's original examples turned out to argue for that model rather than against it.
The `ksamods-gg` field in [content-index#49](https://github.com/KSAModding/content-index/pull/49) is a back-reference to that site's own database record: copying it into the canonical listing creates the second copy, and a freshness problem that the derived view does not have.
The Borea `accent` example never had a clear owner at all — either it is Borea-local presentation data, in which case the derived view holds it, or it is an author-selected presentation field, in which case every client wants it and it is first-class by RFC.

### Ownership has no answer here

RFC 0033 binds write access to a listing to the content author, through the ownership check that [content-index#34](https://github.com/KSAModding/content-index/pull/34) implements for RFC 0048.
Nothing in that model lets a client or an index maintain live data of its own inside somebody else's listing.

So an extension is written by the content author, and every consumer-side update to it has to be submitted by the content author, who has no reason to care and no way to know it changed.
This RFC never answered that, and the answers available are all worse than the derived view: a second write path around the ownership check, or data that is stale by construction.

### The fact it would need does not exist yet

What would justify the table is a fact that is **author-owned**, has **exactly one consumer**, and **must be canonical** — carried by a `git clone` of the index, which is the mirroring guarantee RFC 0033 sells and the one thing a derived view does not provide.

Nobody in review could name one.
Author-owned facts turn out to be wanted by every client, which makes them first-class fields; single-consumer facts turn out to be owned by that consumer, which puts them in its derived view.
Icons and banners are the worked example of the first half: wanted by everyone, and being decided as a `spec_version = 1` gap in [#45](https://github.com/KSAModding/content-manager-design/discussions/45) rather than parked in a namespace.

## Guide-level explanation

What was proposed, recorded for whoever picks this up.

```toml
id = "AdvancedFlightComputer"
type = "mod"
# ... the rest of the authored file

[extensions.borea]
accent = "#ff8800"
```

Each direct child of `[extensions]` is a namespace named after the consumer that owns it.
That consumer reads its own namespace and ignores every other; a client that knows none of them loses nothing.

The line an extension must not cross: it may change how a listing is **presented**, never what is **resolved, downloaded, or installed**.

The name is `[extensions]` rather than `[metadata]` because the entire file is metadata, so the broader word says nothing about what is inside.

## Reference-level explanation

The design as review left it, corrections included.
None of it is normative while this RFC is postponed.

### Where it lives

In the shared authored core of [RFC 0031](0031-content-metadata-format.md), for `mod` and `mod-loader` only.

**`modpack` is excluded.**
A pack document is immutable per version, so an extension on a pack is frozen at the version that carried it rather than live, changing one would require publishing a new pack version, and the full size allowance would repeat for every version in the snapshot.
That contradicts the live-data reason the table exists for, and excluding packs was the smallest of the three ways out.

### Shape

`[extensions]` is a table of tables.

| Rule | Reason |
|---|---|
| Every direct child of `[extensions]` is a table. A scalar or array directly under `[extensions]` makes the file invalid. | Un-namespaced keys are how two consumers end up fighting over `accent`. Forcing the namespace makes a collision impossible rather than unlikely. |
| Namespace names follow RFC 0031's id rules, compared case-insensitively. | Reusing the id charset means no second naming scheme to specify, and it is already known to be safe on every platform. |
| A namespace need not be a listed id, and claiming one grants nothing. | The table names consumers, not content, and `borea` is not a mod. |
| At most 4 KiB per listing, measured as defined below. | Every byte lands in the snapshot RFC 0033 ships to every client, so an unbounded table is a cost paid mostly by people who never read it. |

### How the size is measured

One representation, so the schema check, the snapshot builder and any other implementation compute the same number:

**The UTF-8 byte length of the parsed `extensions` value serialized as compact JSON, with no insignificant whitespace.**

TOML comments, indentation, and key order do not count, and neither does the two-space indentation `spec/snapshot.md` applies when the value reaches the snapshot.
Whether 4 KiB is the right number, and whether it should be per listing or per namespace, was never settled; per listing is simpler and makes namespaces compete for the allowance, which may be the wrong incentive.

### What it may not do

1. **No consumer may read a namespace it does not own.** Reading someone else's namespace makes their private key your public contract, and they will change it.
2. **Nothing under `[extensions]` may affect resolution, dependencies, compatibility, install, load order, or versioning.** Presentation is what it is for; behaviour is what it is not. A validator cannot tell the two apart, so this is a boundary the promotion path has to keep, not something the index can enforce.
3. **It is not stamped into release files.** RFC 0031's `listing` block freezes the display facts a release shipped with; this is live data owned by consumers rather than by the release, and a frozen copy would only go stale.
4. **It is authored text like any other.** RFC 0033's moderation, delisting, and takedown path applies unchanged.

### The snapshot carries it

No new fetch path, and no client-side choice.
`spec/snapshot.md` puts every authored document in the snapshot verbatim, and that document is sufficient for offline use on its own, so an `extensions` table appears there for every listing that carries one.
A separate fetch for selected authored fields would have to amend the snapshot contract and give up its one-fetch and offline guarantees, which no saving here justifies.

### The schema is part of this RFC, not left open

The rules above are normative, so the companion pull request against `KSAModding/content-index` implements exactly them and has no latitude to permit a free subtree at `extensions`: a table, each direct child a table, each name a valid id, case-insensitive duplicates rejected, and the size rule as measured above.
Without that change the field is rejected at publish time regardless of what this RFC says, the way RFC 0048 depends on [content-index#34](https://github.com/KSAModding/content-index/pull/34).

### Promotion

A key that several consumers independently carry is evidence for a first-class field, not a permanent home.
This was the mechanism the table existed to serve, and it is the part worth keeping if anyone reopens the idea.

## Drawbacks

- **It is a place to put things instead of deciding what they are.** Every field that should have been argued through an RFC can be shipped quietly under a namespace, and the promotion path is a convention with nothing enforcing it.
- **The presentation-versus-behaviour line is unenforceable.** Nothing stops a client resolving on `extensions.itself.requires`, and nobody outside that client would know.
- **It reopens `additionalProperties: false`.** The index's schema is closed today and catches typos and junk for free; one permissive subtree is a small hole, but it is the first one.
- **It makes the format partly opaque.** A reader of a listing can no longer tell what all of it means, which is a real loss for a format whose selling point is that one file says everything.
- **Namespaces are unowned.** Anyone can write `[extensions.borea]` into their own listing, so Borea has to treat its own namespace as untrusted input from the author.

## Alternatives

**Do nothing, and use the derived view.**
The accepted model from #37, and what postponing this RFC leaves in place.
A consumer keeps its own data in its own store keyed by listing id, with one copy, one owner, and no freshness problem.

**A first-class field per need.**
Correct, and where facts that more than one consumer wants belong.
The cost is an RFC per field, which is the cost of deciding what something means.

**Relax `[links]` to carry non-URL values.**
Rejected because the URL constraint is the only reason a client can render `[links]` as links without checking each value.

**One flat un-namespaced table.**
Rejected because the first collision is unfixable: two consumers using `accent` for different things cannot both be right, and there is no authority to arbitrate.

## Unresolved questions

Left open, and answering the first is what would reopen this RFC.

1. **Is there an author-owned, single-consumer fact that must be canonical?** Carried by a `git clone` of the index, not reconstructible from a derived view keyed by listing id. One concrete example reopens this; none was found in review.
2. **Is 4 KiB the right cap, per listing or per namespace?**
3. **Should namespaces be registered anywhere?** Unowned is simplest and matches the ecosystem's size; a squatted namespace has no remedy.

## Future possibilities

- **Reopening on evidence**, against unresolved question 1 rather than on the general appeal of an extension point.
- **A registry of known namespaces**, if squatting ever becomes real rather than theoretical.

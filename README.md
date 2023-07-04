# Notes for TUF atop Merkle DAGs

Here are some notes for doing a modified TUF atop some sort of recursive content-addressing system (like IPFS).

## Goals

### Just as secure

This should have all the same security properties as TUF as specified today, and no regressions.

### Leverage Merkle DAG for consistent snapshots

- There should be no way for metadata to get out of sync in the first place, because we must to provide a select few content-address roots by which everything else is reachable.

- Version numbers should not be necessary for consistent snapshots.
  They could still be included for informational purposes, but nothing should require them to exist.

  (For other purposes, we can replace them with a "previous version" reference.)

### No more "repository"

There should be no more notion of a "repository".
Live objects should always be reachable by traversing content addresses, and *only* reachable by traversing content addresses.
(Content addresses can be thought of as capabilities to read something, in addition to making tamper-proof what is ready.)

This is also why sequential version numbers are now insufficient:
version numbers assume there is some total order thus far clear from context.
But Merkle DAG objects (DAG vs tree is important here) do not have a single canonical context.
We have to adopt a more "possible worlds" mindset for how the client might have traversed hashes to reach this object.

### Avoid naming things where possible

The file system requires every file be named, but a recursive content-addressing system allows looking up data by hash.
This means, for example, rather than having a list of names referring to other files (some relative path convention), it is possible to just have a list of hashes which can be looked up directly.
A few opportunities might arise where we can take advantage of this.

However, due to the constraint of [no extra signing](#extra-signing), these opportunities are not as numerous as one might think.

### Mirroring without the mirrors role

This is in fact already supported by regular TUF.

## Non-goals / constraints

### Extra signing

One sentence in the current spec [on consistent snapshots](https://theupdateframework.github.io/specification/latest/#writing-consistent-snapshots) is this:

> This rule is in place so that metadata signed by roles with offline keys will not be forced to sign for the metadata file whenever it is updated.

A design that was too eager to leverage Merkle DAGs might end up making signed objects directly refer to other signed objects.
This would force the parent object to recreated and resigned whenever the child object is updated --- and likewise for the entire ancestor spine up to the root node.
This is bad for security and must not happen.

Like TUF as written, the vast majority signed objects should instead never directly refer to other signed objects.
If we collapse the graph to only account for nodes with signatures ("inlining" child nodes with just "plain old data"), the vast majority of signed objects should be *leaves* in the resulting graph.

In particular, the following roles MUST only sign leaf objects in the previous sense:

- Root
- Targets

The exceptions are:

- Snapshot:

  The entire purpose of the snapshot role, as we shall see, is to "glue" all the other data together so it is reachable from one single root content-address.
  It must therefore sign the root object (not to be confused with what the root role signs!)

- Timestamp:

  The timestamp role signs an ephemeral message.
  This could be within the content-addressing system, but it could also be a "bootstrap" message stored in a separate mutable system.
  In the former case, it is a object with the snapshot's object as its child (and thus the snapshot object is not quite the root object).

# Data model

I'll go through the [documentation formats](https://theupdateframework.github.io/specification/latest/#document-formats) section of the spec and take notes.

This is the data model "at rest", the data model for a single snapshot point in time.
Consistency of the snapshot in isolation is covered as part of the data, but consistency *across* updates (between snapshots) is out of scope and covered in the next section.

### Metaformat --- [original](https://theupdateframework.github.io/specification/latest/#metaformat)

Still true, that is the heart of what we are leveraging for this.

### Object formats: general principles --- [original](https://theupdateframework.github.io/specification/latest/#file-formats-general-principles)

No changes.

### Object formats: root --- [original](https://theupdateframework.github.io/specification/latest/#file-formats-root)

> Note:
> Keep in mind we'll drop all `.json` as we are not using files.
> This is a not a change as this section is just specifying the *contents* of the object, not any *name*.)

#### Previous Version

`"version" : VERSION` is replaced with a `previous-version` attribute to an optional content address pointing to the previous version of the object.
The previous version content address chain induces a total order on versions of this object.

In IPFS this could be:

```json5
{
  "previous-version" : null, // original root object
  "previous-version" : { "/": CID<Root> }, // IPLD link
}
```
(See https://ipld.io/docs/data-model/kinds/#link-kind)

### Object formats: snapshot --- [original](https://theupdateframework.github.io/specification/latest/#file-formats-snapshot)

#### Previous version

The (top-level) `version` attribute should be replaced with a `previous-version` attribute [just like for the root object](#previous-version).

#### Meta

This object should be overhauled to be more semantic, and leverage the ambient content-addressing system.

- Drop all `.json`

  We are no longer referring to files.

- Also point to the root object

  Since there is no notion of a repository this the *only* way the root object is found.

- Don't list delegated targets at the top level

  The other roles are "fixed", only delegated roles have arbitrary names.
  Therefore, delegated roles should be put in a child map.
  
  > This abides by the general well-typing principles that maps (arbitrary keys, homogeneous values) and records (fixed keys, heterogeneous values) should never be mixed up.

- Replace `METAPATH` with a content address

  The details of content addresses are the domain of the underlying system.

#### Example

The example, with changes, adapted for IPFS, would be:

```json5
{
  "signatures": [
    {
      "keyid": "66676daa73bdfb4804b56070c8927ae491e2a6c2314f05b854dea94de8ff6bfc",
      "sig": "f7f03b13e3f4a78a23561419fc0dd741a637e49ee671251be9f8f3fceedfc112e4
              4ee3aaff2278fad9164ab039118d4dc53f22f94900dae9a147aa4d35dcfc0f"
    }
  ],
  "signed": {
    "_type": "snapshot",
    "spec_version": "1.0.0",
    "expires": "2030-01-01T00:00:00Z",
    "meta": {
      "root": { "/": CID<Root> },
      "targets": { "/": CID<Target> },
      "delegated-targets: {
        "project1": { "/": CID<Target> },
        "project2": { "/": CID<Target> },
      },
    },
    "previous-version": null,
  }
}
```

#### Self-Consistency

All roles must be hierarchy delegated from the root:

- `root` authorizes `snapshot`

  > Note how the child object is authorizing the parent object.
  > This means that, unfortunately, we must process the child object before we know that we trust it.
  > This is generally a bad thing in cryptosystems, but it is an inevitable consequence of the fact that the root role is delegating authority in the sense that it doesn't want to sign everything signed by a delegated role.
  > Clients are therefore strongly encouraged to get `root` object from the snapshot and validate it before processing any other part of the snapshot.

- `root` authorizes `targets`

- `targets` authorizes some roles within `delegated-targets`

- Some of `delegated-targets` authorizes rest of `delegated-targets`

  In particular, from `targets` and then recursively following delegations, all members of `delegated-targets` MUST be reachable.

### Target Object --- [original](https://theupdateframework.github.io/specification/latest/#file-formats-targets)

TODO

### Timestamp Object --- [original](https://theupdateframework.github.io/specification/latest/#file-formats-timestamp)

#### Drop version field

The version field is not needed.
The authority a timestamp must be established by traversing to the snapshot and then to the root object, and checking the signature.
By that point, the snapshot object is known, and its `previous-version` field will suffice.


#### Single snapshot not metadata map

We only need to refer to one object, and we can just do so by content address.
This means that all the extra structure is not needed.
Having a map but having that map must only has one element implies a degree of freedom that doesn't in fact exist.

Instead, have a single attribute `snapshot` with the content address of the snapshot object.

### Example

The example, with changes, adapted for IPFS, would be:

```json5
{
  "signatures": [
    {
      "keyid": "1bf1c6e3cdd3d3a8420b19199e27511999850f4b376c4547b2f32fba7e80fca3",
      "sig": "90d2a06c7a6c2a6a93a9f5771eb2e5ce0c93dd580bebc2080d10894623cfd6eaed
              f4df84891d5aa37ace3ae3736a698e082e12c300dfe5aee92ea33a8f461f02"
    }
  ],
  "signed": {
    "_type": "timestamp",
    "spec_version": "1.0.0",
    "expires": "2030-01-01T00:00:00Z",
    "snapshot": { "/": CID<Snapshot> }
  }
}
```

## Cross-version consistency checks

The consistency of a single snapshot is formalized above; we might call this *spatial* consistency.
But for upgrades to be trustworthy, we must also define a notion of *temporal* consistency.

Temporal consistency is a relation between two objects of the same sort, answering whether it is valid to update to the first from the second.
We want such a relation on for every type of object, and to define it inductively on the structure of that type.

### Notation

We'll use [Natural Deduction](https://en.wikipedia.org/wiki/Natural_deduction) as it is a widely-used notation for defining relations by induction.

### Relations

- `VALID A`: A is self-consistent
- `A <- B`: A is the immediate successor to B
- `A <-? B`: A is either B or the immediate successor to B
- `A <-* B`: A is a reflexive-transitive successor to B

It should be derived (not assumed) that the second two binary relations only relate elements that are themselves valid.

### Inductive rules

#### Any `<-?`

- Reflexivity
  ```
  VALID ObjectA
  ----------------
  ObjectA <-? ObjectA
  ```

- Single step
  ```
  ObjectA <- ObjectB
  ----------------
  ObjectA <-? ObjectC
  ```

#### Any `<-*`

- Reflexivity
  ```
  VALID ObjectA
  ----------------
  ObjectA <-* ObjectA
  ```

- Transitivity
  ```
  ObjectA <- ObjectB
  ObjectB <-* ObjectC
  ----------------
  ObjectA <-* ObjectC
  ```

#### Root Object `<-`

- Root single step
  ```
  VALID RootA
  VALID RootB
  RootA.previous-version = RootB
  thresholdSign(RootA, RootB)
  ----------------
  RootA <- RootB
  ```
  `thresholdSign` following https://theupdateframework.github.io/specification/latest/#key-management-and-migration.

#### Targets Object `<-`

TODO

#### Snapshot Object `<-`

- Snapshot single step
  ```
  VALID SnapshotA
  VALID SnapshotB
  SnapshotA.previous-version = SnapshotB
  SnapshotA.root <-? SnapshotA.root
  SnapshotA.targets <-? SnapshotA.targets
  for all keys in both maps:
    SnapshotA.delegated-targets[key] <-? SnapshotB.delegated-targets[key]
  ----------------
  SnapshotA <- SnapshotB
  ```

Notes:

- Delegated targets that are only in one or the other snapshots are fine.
  This reflects that they can be added and removed at will.
  `VALID` ensures that there are no unreachable or missing delegates.

- Using `<-?` in inductive hypotheses is requiring that snapshots cannot "skip steps".
  Every time any of the underlying objects changes, we must make sure a snapshot is created.
  Snapshot can batch changes, however.

#### Timestemp Object `<-`

  ```
  VALID TimestampA
  VALID TimestampB
  TimestampA.snapshot <-* TimestampB.snapshot
  ----------------
  TimestampA <- TimestampB
  ```

Notes:

- We use `<-*` is used in the inductive hypothesis (above the line) to reflect the fact that we do not need or require the full history of snapshots.
  We can only care about `<-*` on timestamps, which this amends itself to.
  I.e. we can grab any two timestamps, and just worry about establishing the previous-version provenance chain between the snapshots they point to.

#### TODO: Gaps in formalization

Probably these things are in parts of the TUF spec I have not read yet :).

- Unclear how the `expires` attribute is used and not used.
  (e.g. when rolling up the history on old objects.)
  When does it matter that the previous version hasn't expired, and when does it not?
  Perhaps only for root object, when one needs to sign the next while it is still valid?
  I think all other authorization is "spatial" not "temporal".

- Is it OK to allow sparse snapshot history unlike what is written?
  I.e. Is it OK to allow snapshots to skip version in the underlying objects?
  Maybe even the root object?!

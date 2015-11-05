title: "Technical specification of repository signing and patches"
authors: [ "Hannes Mehnert" {"mailto:hannes(à)mehnert.org"} ]
date: "2015-10-30 00:00 UTC+0900"
--BODY--

This document describes the technical details of the signing proposal, and is
independent of OPAM and git.  Not in scope is the distribution of the initial
set of trusted keys (`TK`).

A repository is a directory consisting of files and directories.  Any kind of
links are prohibited.

The repository can evolve in multiple ways: new keys can be introduced who can
add subdirectories, which are then owned by them (they can delegate permissions,
add more files and subdirectories).  A (transitive) quorum of `TK` has ownership
over the entire repository and can modify arbitrarily.  Without `TK`, the
repository can end up in containing subdirectories not owned by any key anymore.

The claim of this document is that a repository fulfills at every given moment
the invariant that all of its content can be verified if `TK` is trusted.  Each
update preserves this invariant.  Since `TK` is mirrored inside of the
repository, revocation of these keys are possible (if a quorum of them remain
valid, if not a new `TK` has to be distributed to all clients).


## Repository layout

The repository contains apart from data also metadata, namely keys, ownership
information (delegations), and checksums of subdirectories.  Therefore we impose
some structure onto the directory:
- A `keys` directory with one signed file per key, named as the key identifier.
- Each directory (apart from '/' and 'keys') may contain a signed file
  `delegate`, which contains ownership information.
- Each directory with data (or which used to contain data) contains a signed
  `checksums` file.  This lists all files in the current directory and its
  subdirectories with their checksum.

Since `keys`, `delegate` and `checksums` are signed, they do not appear in any
`checksums` file.  To prevent rollback attacks, each signed file contains a
monotonic counter.

An example layout is:
```
repository root /
|-- bla/
|   |-- a/
|   |   |-- delegate
|   |   |-- a.1/
|   |   |   |-- a
|   |   |   |-- b
|   |   |   |-- c
|   |   |   |-- dir/
|   |   |   |   `-- d
|   |   |   `-- checksums
|   |   `-- a.2/
|   |       `-- checksums
|   `-- b/
|       `-- delegate
`-- keys/
    |-- developer1
    `-- developer2
```

### Signature

A signature is encoded as a triple `(keyid, signature-algorithm, value)`.

- `keyid` is the key identifier, a string
- `signature-algorithm` is a string
- `value` is the base64-encoded signature.

### Keys

A public key is a PEM-encoded (of a DER-encoded pkInfo structure, including an
AlgorithmIdentifier and the actual public key) key.  For the remainder of this
document we will assume RSA keys, but it should not matter which public key
algorithm is used.

A public key is defined by a quadruple `(counter, keyid, key, signatures)`.

- `counter` is a monotonic counter starting from 0
- `keyid` is a unique identifier (printable ASCII characters, no control
  characters), the contained file has the same name as the keyid.  Uniqueness
  over case-insensitive keyids must be preserved throughout the repository.
- `key` is a pem-encoded key, or the empty string (revoked key)
- `signatures` is a list of signatures

### Delegation

The purpose of a `delegate` file is to contain a list of key identifiers which
own the contained directory.  A `delegate` file consists of a quadruple
`(name, counter, key-ids, signatures)`.

- `name` is the path from the repository root to here, encoded as string
- `counter` is a monotonic counter starting from 0
- `key-ids` is a set of key identifiers who own this directory
- `signatures` is a list of signatures

### Checksums file

A `checksums` file contains checksums of all files in the containing directory,
and subdirectories thereof.  It is a quadruple
`(counter, name, files, signatures)`.

- `counter` is a monotonic counter
- `name` is the path from the directory root to here, encoded as string
- `files` is a list of triples `(filename, byte-size, sha256-digest)`
- `signatures` a list of signatures


## Repository invariant

Given a set of trusted keys (`TK`) and the concrete quorum connected to the
repository, we can verify a repository and any given time.  To do so, we first
create the acyclic graph of signatures on keys, and sort it topologically using
the `signs` relation.  We validate all signatures on all keys, and split them
into two sets, the repository maintainers, which have a quorum of signatures
originating in `TK`, and the developers, which are at least self-signed.

Now, each `delegate` file has to be signed either by a quorum of repository
maintainers or by a developer listed in itself.  Each data directory needs to
contain a `checksums` file, which has all data files listed, and is signed
either by an owner (using the list in the `delegate` file), or by a quorum of
repository maintainers.

Key files can contain the empty key, delegates the empty list, checksums the
empty list of files.  These are necessary for revoked keys and deleted content.


## Repository modifications

A modification is a patch.  Each patch can be verified individually: Given a
repository in state `S`, and a patch `P`, the repository evolves into state
`S'`, which is `S` with `P` applied.  In both states `S` and `S'` the invariant
holds.

Each patch contains an ordered list of hunks. Each hunk can be classified into
one of three categories, for each apply different verification rules: key
modification, delegate modification, data modification.  Only a single hunk
which modifies a key is allowed in a patch, and is verified first.  (There might
be subsequent hunks which change signatures of other keys, though.)  Afterwards,
delegate modifications are verified, and last data modifications.

Hunks (h<sub>n</sub>) are processed in order, leading to `S` &rarr;<sub>h<sub>1</sub></sub> `S`<sub>`1`</sub> &rarr;<sub>h<sub>2</sub></sub> `S`<sub>2</sub> ... &rarr;<sub>h<sub>n-1</sub></sub> `S`<sub>n-1</sub> &rarr;<sub>h<sub>n</sub></sub> `S`<sub>n</sub> where `S'` (mentioned above) is `S`<sub>n</sub>.

There are two different cases where a file modification is valid (`mod_valid`):
- _either_ a key responsible for this file signed it
- _or_ a quorum of keys signed the file. Each key participating in the quorum
  has to be signed by a quorum of keys itself, ultimately transitively rooted
  in `TK`.

Each hunk is verified in the following way (using repository state `S` and
`TK`), depending on its content:
- key modification:
   - if it adds a new key file:
     - key is big enough
     - `keyid` is unique and matches filename
     - `counter` is 0
     - file is self-signed
   - if it modifies an existing key:
     - `counter` increased
     - `keyid` matches filename
     - `mod_valid` (see above) either key in `S` signed, or a quorum of `TK`
   - deletion of a key is invalid
- `delegate` modification:
   - addition of a delegate is signed by a keyid in the `key-ids` list
   - deletion of a delegate is invalid
   - `counter` field increased (or 0 if addition)
   - modification: `mod_valid` signed by a key of the list `key-ids` in state
     `S`, or a quorum of `TK`
- data modification:
   - find the closest `delegate` file between `d` and `/` in `S`
   - `K` is the set of `key-ids` of this `delegate` file, else empty
   - `counter` field in the `checksums` file increased (or 0 if addition)
   - `checksums` is `mod_valid` by any key in `K` or a quorum of `TK`
   - all files in the directory and its subdirectories occur in `files`, and
     have the correct length and checksum.

## Concrete instantiation (what is in Code)

- `signature-algorithm` can be one of the following, as described in [PKCS1][]:
   - "RSA-PSS" (using RSASSA-PSS: improved probabilistic signature scheme with
     appendix; based on the Probabilistic Signature Scheme originally invented
     by Bellare and Rogaway)
   - "RSA-PKCS" (using RSASSA-PKCS1-v1_5: old signature scheme with appendix as
     first standardized in version 1.5 of PKCS #1).

- instead of doing the transitive quorum check to split keys into repository
  maintainers and developers, we introduced a `role` field in each key, which is
  a string.  If it is anything apart from `Developer`, a quorum of RMs need to
  have signed this key.

[PKCS1]: https://tools.ietf.org/html/rfc3447

## Common processes

- introduce a new key, self-signed
- introduce a new RM (change role): requires quorum
- rollover a key k: change k to k', sign with k, sign with k', sign everything signed with k with k'
- revocation of a key: empty file, requires quorum
- revocation of a RM: change role, requires quorum, ensure all signatures made with RM role are either resigned by some other RM or "removed"

## Difference to TUF target and delegation role

We want to preserve the invariant that once something is owned by a key, it
stays owned by that key.  Therefore we track patches (since delegation is not
signed by RMs).

Python uses `claimed` vs `unclaimed` names instead, which we might want to
adapt (by requiring quorum of signatures on `delegate` file for a `claimed`
name).

And we have a git-like repository in mind, rather than a directory.  Thus
several people should be able to change subparts (which they own) of the system
in a concurrent way, without merge conflicts.  This is the reason why we
distribute metadata (keys, delegate, checksums) in a per-identity, per-package
wat, instead of having a single global file.

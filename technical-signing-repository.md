title: "Technical specification of repository signing and patches"
authors: [ "Hannes Mehnert" {"mailto:hannes(à)mehnert.org"} ]
date: "2015-10-30 00:00 UTC+0900"
--BODY--

This document describes the technical details of the signing proposal, and is
rather independent of OPAM and git.

A repository is simply a directory, symbolic or hard links are prohibited.

In this document, I argue that this repository can only be modified by
(transitive) trusted (delegated) keys.

## Repository layout

The repository has some structure: it contains a `keys/` directory with a flat
structure of keys.  Other subdirectories may either contain a `delegate` file
and a set of subdirectories, or a `checksums` file,

Each key is named after its key identifier, and self-signed.

An example layout would be:
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

### Keys

A public key is a PEM-encoded (of a DER-encoded pkInfo structure, including an
AlgorithmIdentifier and the actual public key) key.  For the remainder of this
document we will assume RSA keys, but it should not matter which public key
algorithm is used.

A public key is thus defined by a quadruple:
`(counter, keyid, key, signatures)`

- `counter`: monotonic counter
- `keyid`: a unique identifier (printable ASCII string, no control characters),
  the filename must be the same value.  Uniqueness (case-insensitive) between
  all keys must be preserved.
- `key`: as described above, or the empty string if the keyid is revoked
- `signatures`: a list of signatures (defined below), at least a self signature!

### Signature

A signature is encoded as a triple:
`(keyid, signature-algorithm, value)`

- `keyid` is a string
- `signature-algorithm` is a string (described in [PKCS1][]):
   - "RSA-PSS" (using RSASSA-PSS: improved probabilistic signature scheme with
     appendix; based on the Probabilistic Signature Scheme originally invented
     by Bellare and Rogaway)
   - "RSA-PKCS" (using RSASSA-PKCS1-v1_5: old signature scheme with appendix as
     first standardized in version 1.5 of PKCS #1).
- `value` is the base64 encoded signature.

### Delegation

The purpose of a `delegate` file is to contain a list of key identifiers which
have permission to modify the contained directory.  A `delegate` file consists
of a quadruple: `(name, counter, key-ids, signatures)`:
- `name` is the directory name
- `counter` is a monotonic counter
- `key-ids` is a set of key identifiers
- `signatures` defined below

### Checksums file

A `checksums` file consists of checksums of (recursively!) all files in the
directory.  It is a quadruple
`(counter, name, files, signatures)`:

- `counter` is a monotonic counter
- `name` the directory name
- `files` is a list of triples `(filename, byte-size, sha256-digest)`
- `signatures` a list of signatures

## Repository, and evolution

Let us start with the empty repository `E`.  This is trivially safe and trusted.

We also need an initial set of trusted public keys, named `TK`.  These are
distributed via a second channel (e.g. bundled with the application, printed and
mailed, publicly available and signed by some well-trusted OpenPGP keys).

Each modification to the repository, a so-called `patch`, is verified
individually:  Given a repository in state `S`, and a patch `P`, the repository
evolves into state `S'`, which is `S` with `P` applied.

There are two different cases where a file modification is valid:
- a key responsible for this file signed it OR
- a quorum of keys signed the file. Each key participating in the quorum has to
  be signed by a quorum of keys itself, ultimately transitively rooted in `TK`.

A patch `P` consists a set of modifications to files.  The patch is split into
components: a potential modification of a single file in `keys/` is processed
first, followed by individual modifications to `delegate` files, followed by
modifications grouped by subdirectory including a `checksums` file.

Components (c_n) are processed in order, leading to
`S` -(c_1)&rarr; `S_1` -(c_2)&rarr; `S_2` ... -(c_(n-1))&rarr; `S_(n-1)` -(c_n)&rarr; `S_n`
where `S'` mentioned earlier is `S_n`.

Finding the closest file `f` between `x` and `y`, where `y/.../x`:
```
find f x y =
  if x = y then
    None
  else
    if (exists x/f) then
      Some x/f
    else
      find f x/.. y
```

Each component is verified in the following way, depending on its content:
- a single key file:
   - if it adds a new file: contains a valid key, which is unique, and the file
     has a valid self-signature
   - if it modifies an existing key:
     - verify that `counter` increased
     - either the public key of the keyid in `S` signs this file, or a quorum
       rooted in `TK`.
   - any deletion of a key is invalid
- each `delegate` file:
   - any deletion is invalid
   - the `counter` field increased
   - load key identifiers from `S` into `K`, and use them to verify the
     signatures of `delegate`: either signed by a key in `K`, or by a quorum of
     keys rooted in `TK`
- other modifications in some directory `d`:
   - find `delegate` file between `d` and `/` in `S` (using algorithm above)
   - `K` is the set of all key identifiers of this `delegate` file, else empty
   - the `counter` field in the `checksums` file increased
   - apply the modifications, and verify that the checksums of all files are
     correct.  Only exactly those files listed in `checksums` have to exist
     (plus the `checksums` file itself)!
   - verify the signatures of the `checksums` file: either one key of `K` or a
     quorum rooted in `TK`

[PKCS1]: https://tools.ietf.org/html/rfc3447

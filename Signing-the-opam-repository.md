title: "Signing the OPAM repository"
authors: [ "Louis Gesbert" {"mailto:louis.gesbert(à)ocamlpro.com"}
           "Hannes Mehnert" {"mailto:hannes(à)mehnert.org"} ]
date: "2015-06-05 15:41 UTC+0900"
--BODY--

> This is an initial proposal on signing the OPAM repository. Comments and
> discussion are expected on the
> [platform mailing-list](http://lists.ocaml.org/listinfo/platform).

The purpose of this proposal is to enable a secure distribution of
OCaml packages. The package repository does not have to be trusted if
package developers sign their releases.  Each opam file will contain a
cryptographic signature (in a `signature:` field) of its developer,
which includes metadata of all involved files (such as a checksum of
the release tarball).

Like [Python's pip][python-tuf], [Ruby's gems][ruby-tuf] or more recently
[Haskell's hackage][haskell-tuf], we are going to implement a flavour of The
Upgrade Framework ([TUF][tuf]). This is good because:

- it has been designed by people who [know the stuff][thandy] much better than
  us
- it is built upon a threat model including many kinds of attacks, and there are
  some non-obvious ones (see the [specification][tuf-spec], and below)
- it has been thoroughly reviewed
- following it may help us avoid a lot of mistakes

Importantly, it doesn't enforce any specific cryptography, allowing us to go
with what we have [at the moment][nocrypto] in native OCaml, and evolve later,
_e.g._ by allowing ed25519.

There are several differences between the goal of TUF and opam, namely
TUF distributes a directory structure containing the code archive,
whereas opam distributes metadata about OCaml packages. Opam uses git
(and GitHub at the moment) as a first class citizen: new packages are
submitted as pull requests by developers who already have a GitHub
account.

Note that TUF specifies the signing hierarchy and the format to deliver and
check signatures, but allows a lot of flexibility in how the original files are
signed: we can have packages automatically signed on the official repository, or
individually signed by developers. Or indeed allow both, depending on the
package.

Below, we tried to explain the specifics of our implementation, and mostly the
user and developer-visible changes. It should be understandable without prior
knowledge of TUF.

We are inspired by [Haskell's adjustments][haskell-tuf-git] (and
[e2e][haskell-tuf-e2e]) to TUF using a git repository for packages. A
signed repository and signed packages are orthogonal. In this
proposal, we aim for both, but will describe them independently.

## Threat model

- An attacker can compromise at least one of the package distribution
  system's online trusted keys.

- An attacker compromising multiple keys may do so at once or over a
  period of time.

- An attacker can respond to client requests (MITM or server
  compromise) during downloading of the repository, a package, and
  also while uploading a new package release.

- An attacker knows of vulnerabilities in historical versions of one or
  more packages, but not in any current version (protecting against
  zero-day exploits is emphatically out-of-scope).

- Offline keys are safe and securely stored.

An attacker is considered successful if they can cause a client to
build and install (or leave installed) something other than the most
up-to-date version of the software the client is updating. If the
attacker is preventing the installation of updates, they want clients
to not realize there is anything wrong.

## Attacks

- Arbitrary package: an attacker should not be able to provide a package
  they created in place of a package a user wants to install (via MITM
  during package upload, package download, or server compromise).

- Rollback attacks: an attacker should not be able to trick clients into
  installing software that is older than that which the client
  previously knew to be available.

- Indefinite freeze attacks: an attacker should not be able to respond
  to client requests with the same, outdated metadata without the
  client being aware of the problem.

- Endless data attacks: an attacker should not be able to respond to
  client requests with huge amounts of data (extremely large files)
  that interfere with the client's system.

- Slow retrieval attacks: an attacker should not be able to prevent
  clients from being aware of interference with receiving updates by
  responding to client requests so slowly that automated updates never
  complete.

- Extraneous dependencies attacks: an attacker should not be able to
  cause clients to download or install software dependencies that are
  not the intended dependencies.

- Mix-and-match attacks: an attacker should not be able to trick clients
  into using a combination of metadata that never existed together on
  the repository at the same time.

- Malicious repository mirrors: should not be able to prevent updates
  from good mirrors.

- Wrong developer attack: an attacker should not be able to upload a new
  version of a package for which they are not the real developer.

## Trust

A difficult problem in a cryptosystem is key distribution.  In this proposal, a
package developer signs their new release, submits a new package, delegates
responsibility for a package, renew their key, ...
Some operations need interaction with a group of janitors, the repository
maintainers, responsible for keeping the repository in a working state
(by adjusting version constraints, helping with key revocation, preventing
malicious individuals from squatting all package names, ...).

### End-to-end signing

The end-user must be able to validate a signature made by the original
developer. The trust path for the chain of trust (where
"&rarr;" stands for "signs for"):

- repository maintainer keys &rarr;
  snapshot key &rarr;
  package delegation + developer key &rarr;
  package files

It must be easy enough for new developers to publish their packages.
When a developer releases a package, they sign the new release with
their public key and submit a pull request.

### Repository maintainer (RM) keys

An initial set of repository maintainer keys for the official opam OCaml
repository is distributed with each opam release.  The private keys will be
stored password-encrypted, securely offline.

The responsibility of the RM is to keep the repository in a working state:
introducing version constraints of reverse dependencies upon new releases,
taking care of package names (and key identifiers), adding new members to the
group of RMs, removing members of the group, revoking compromise keys, ...

If a single repository maintainer key would be valid for modifying every
package, they would be too easy targets for compromises.  Instead, each
operation needs a quorum of at least three repository maintainers.  The quorum
has several benefits:

- the compromise of a number of RM keys less than the quorum is harmless
- it allows to safely revoke and replace a key, even if it was lost

The initial RM keys could be distributed as such:
- Louis Gesbert, opam maintainer, OCamlPro
- Anil Madhavapeddy, main repository maintainer, OCaml Labs
- Thomas Gazagnaire, main repository maintainer, OCaml Labs
- Grégoire Henry, OCamlPro safekeeper
- Someone in the OCaml team ?
- Other maintainers of the ocaml/opam-repository GitHub repository

For other repositories, there will be three options:
- no signatures (backwards compatible ?), _e.g._ for local network repositories.
  This should be allowed, but with proper warnings.
- trust on first use: get the RM keys on first access, let the user confirm
  their fingerprints, then fully trust them.
- let the user manually supply the RM keys.

### Developer keys

These keys are used by the package developers for end-to-end signing. They are
generated locally as needed by new packagers (_e.g._ by the `opam-publish`
tool), and should be stored password-encrypted. They can be added to the
repository through pull-requests (self-signed), waiting to be signed (i) as part
of snapshots (which also prevents them to be modified later, but we'll get to
it) and (ii) directly by RMs.

Each package directory in opam will include a `delegate` file, which specifies
the list of developers which maintain this package.  When publishing a new
package, the developer includes a (signed by themself) `delegate` file where
at least the developer is listed as maintainer.  At any time, developers can
decide to add and remove new maintainers by modifying the `delegate` file.

### Repository signing

This provides consistent, up-to-date snapshots of the repository, and protects
against a whole different class of attacks than end-to-end signing (_e.g._
rollbacks, mix-and-match, freeze, etc.)

This is done automatically by a snapshot bot (might run on the repository
server), using the _snapshot key_, which is signed by some RM keys,
hence the chain of trust:

- repository maintainer keys &rarr;
  snapshot key &rarr;
  commit-hash

Where "commit-hash" is the head of the repository's git repository (and thus a
valid cryptographic hash of the full repository state, as well as its history)

The snapshot bot is an online automated service which periodically updates its
repository checkout, checks patches and linearity constraints, and signs the
repository with a timestamp.  The snapshot key has lower security than the RM
keys, but also a lower potential: it can not be used to sign packages, and thus
inserting malicious code or metadata is not possible.

### Trust chain and revocation

Opam is distributed with some RM keys, and the opam client has a local
repository (in some commit `C`).  Local state, which includes valid RM keys,
is preserved in `.opam/keystate`.  To update the repository, all commits between
`C` and `HEAD` are fetched, the most recent is expected to be done by the
snapshot bot. First, the validity of the snapshot key is verified by validating
its signatures (which might involve reading and validating of repository
manager keys, but they have to be rooted in the locally preserved RM keys or
those distributed with opam).

Public keys are stored in the repository, each in a file `keys/<id>` (where
`<id>` might be a mail address, or the GitHub user account), self-signed, maybe
signed by RMs, and implicitly signed by the snapshot bot.

Revocation of keys is done by replacing the specific key with the empty string.
This needs to be signed by a quorum of RMs.  For key renewal, the old key is
replaced with the new, and signed by both old and new, in addition to a quorum
of RMs (if RM key, otherwise just self-signed by the developer).

Name squatting of package names and key identifiers is possible, in the case
the MITM is between GitHub and the snapshot bot.

## File formats and hierarchy

### Signed files and tags

The files follow the opam syntax: a list of _fields_ `fieldname:` followed by
contents. The format is detailed in [opam's documentation][opam-format].

The signature of files in opam is done on the canonical textual representation,
following these rules:
- any existing `signatures:` field is removed
- one field per line, ending with a newline
- fields are sorted lexicographically by field name
- newlines, backslashes and double-quotes are escaped in string literals
- spaces are limited to one, and to these cases: after field leaders
  `fieldname:`, between elements in lists, before braced options, between
  operators and their operands
- comments are erased
- fields containing an empty list, or a singleton list containing an empty
  list, are erased

The `signatures:` field is a list with elements in the format of string triplets
`[ "<id>" "<algorithm>" "<signature>" ]`, where the signature is hex encoded.
For example:

```
opam-version: "1.2"
name: "opam.1.3"
signatures: [
  [ "louis.gesbert@ocamlpro.com" "RSA-PSS" "048b6fb4394148267df..." ]
]
```

The snapshot bot creates an annotated tag `signed`, which also uses opam syntax.
It contains a timestamp (UTC, RFC3339 format), a commit-hash, and a `signatures`
field:

```
last-updated: "2015-06-04T00:00:00Z"
commit-hash: "5ead357e18059634e167d107859781002c68b206"
signatures: [ [ "SNAP-foo@bar" "RSA-PSS" "6e817eb01a..." ] ]
```

This `signed` tag is pushed to the opam repository, which is fetched by the
client.

### File hierarchy

The repository format is changed by the addition of:
- a directory `keys/` at the root
- delegation files `packages/<pkgname>/delegate`
- signed checksum files at `packages/<pkgname>/<pkgname>.<version>/signature`

Here is an example:

```
repository root /
|--packages/
|  |--pkgname/
|  |  |--delegate              - signed by developer1, developer2
|  |  |--pkgname.version1/
|  |  |  |--opam
|  |  |  `--signatures         - signed by developer2
|  |  `--pkgname.version2/
|  |     |--files/aa.patch
|  |     |--opam
|  |     `--signatures         - signed by developer1 (incl. aa.patch checksum)
|  `--pkgname2/ ...
`--keys/
   |--developer1               - signed by developer1
   `--developer2               - signed by developer2
```

Keys are provided in individual files, named as `<id>`, in opam format:
```
last-updated: "2015-06-04T00:00:00Z"
keyid: "test@example.com"
role: "developer"
key: "{PEM-encoded public key}"
signatures: [ [ "test@example.com" "RSA-PSS" "6e817eb01a..." ] ]
```

Role is the claimed role, a set of signatures is required to actually act as
this role (always self-signed; a single RM for developer, a quorum for other
roles).  An empty `key` field denotes deletion.

#### Delegation

`/packages/pkgname/delegate` delegates ownership on versions of package
`pkgname`. The file contains version constraints associated with keyids, _e.g._:

```
last-updated: "2005-04-02T12:12:12Z"
name: "{pkgname}"
delegation: [
  "thomas@gazagnaire.org"
  "louis.gesbert@ocamlpro.com" {>= "1.0"}
]
signatures: [ ... ]
```

The file is always signed by some developer in the delegation list; or a quorum
of RMs (for hot-fixes or if developers managed to lock themselves out).

The `delegates:` field may be empty: in this case, no packages by this name are
allowed on the repository. This is used to mark deletion of a package.

#### Package signature files

These guarantee the integrity of a package: this includes metadata and the
package archive itself (which may, or may not, be mirrored on the the opam
repository server).

The file, besides the package name and version, has a field `package-files:`
containing a list of files below `packages/<pkgname>/<pkgname>.<version>`
together with their file sizes in bytes and one or more hashes, prefixed by their
kind, and a field `archive:` containing the same details for the upstream
archive. For example:

```
last-updated: "2003-01-01T23:24:59Z"
name: pkgname.version
package-files: [
  "opam" {901 [ sha1 "7f9bc3cc8a43bd8047656975bec20b578eb7eed9" md5 "1234567890" ]}
  "files/ocaml.4.02.patch" {17243 [ sha1 "b3995688b9fd6f5ebd0dc4669fc113c631340fde" ]}
]
archive: [ 908460 [ sha1 "ec5642fd2faf3ebd9a28f9de85acce0743e53cc2" ] ]
signatures: [ ... ]
```

This file is signed either:
- by a delegate key from the `../../delegate`
- by a quorum of repository maintainers

The latter is needed to hot-fix packages on the repository: repository
maintainers often need to do so. A quorum is required to prevent a single RM key
compromise from allowing arbitrary changes to every package.

If the delegation or signature can't be validated, the package or compiler is
ignored. If any file doesn't correspond to its size or hashes, it is ignored as
well. Any file not mentioned in the signature file is ignored.

## Snapshots and linearity

### Main snapshot role

The snapshot bot periodically adds a `signed` annotated tag to the top of the
served branch of the repository. This tag contains the commit-hash and the
current timestamp, effectively ensuring freshness and consistency of the full
repository. This protects against mix-and-match, rollback and freeze attacks.

The `signed` annotated tag is deleted and recreated by the snapshot bot
periodically, after checking the validity of each individual commit.

Each client when updating fetches the remote repository, validates the `signed`
tag, and each commit individually, and updates its local repository.

### Linearity

The repository is served using git: this means, not only the latest version, but
the full history of changes are known. This as several benefits, among them,
incremental downloads "for free"; and a very easy way to sign snapshots. Another
good point is that we have a working full OCaml implementation.

We mentioned above that we use the snapshot signatures not only for repository
signing, but also as an initial guarantee for submitted developer's keys and
delegations. One may also have noticed, in the above, that we sign for
delegations, keys etc. individually, but without a bundle file that would ensure
no signed files have been maliciously removed.

These concerns are all addressed by a _linearity condition_ on the repository's
git: the snapshot bot does not only check and sign for a given state of the
repository, it checks every individual change to the repository since the last
well-known, signed state: patches have to follow from that git commit
(descendants on the same branch), and are validated to respect certain
conditions: no signed files are removed or altered without signature, etc.

Moreover, this check can also done on clients, every time they update: it is
slightly weaker, as the client don't update continuously (an attacker may have
rewritten the commits since last update), but still gives very good guarantees.
Low-powered clients might just trust the snapshot bot enough to only verify the
single signature.

A key and delegation that have been submitted by a developer and merged, are
signed as part of a snapshot: git and the linearity conditions allow us to
guarantee that this delegation won't be altered or removed afterwards. Even if
the repository is compromised, an attacker won't be able to roll out malicious
updates breaking these conditions to clients.

The linearity invariants are:
 1. no key, delegation, or package version (signed files) may be removed
 2. a new key is signed by itself, and the RM who merged it
 3. a new delegation is signed by the delegate key, and the RM who merged it
 4. a new package or package version is signed by a valid key holding a valid
    delegation for this package version
 5. keys can only be modified with signature from the previous key or a quorum
    of RM keys
 6. delegations can only be modified with signature by delegate keys, or a
    quorum of RM keys
 7. any package modification is signed by an appropriate delegate key, or a
    quorum of RM keys
 8. modifications of any file with a `last-updated` field needs to increase

In case the linearity invariant check fail:
- on the GitHub repository, this is marked and the RMs are advised not to merge
  (or to complete missing tag signatures)
- on the clients, the update is refused, and the user informed of what's going
  on (the repository has likely been compromised at that point)
- on the repository (checks by the snapshot bot), update is stalled and all
  repository maintainers immediately warned. To recover, the broken commits
  (between the last valid `signature` and master) need to be amended.

### How to deal with a compromised repository

If several RM keys have been compromised, and the repository includes unintended
commits, we need a mechanism to get rid of these commits, and return into a
working state.

How to handle this depends on where the compromise was detected: if the snapshot
bot detected a potential compromise, all is fine (since no client was ever
exposed to a compromised repository).

If clients received and used a compromised repository, since amending commits
would violate linearity, users need to be involved to move forward.  Any builtin
mechanism to violate linearity could be misused by an attacker.  The opam RMs
publish a statement of the compromise, and sign it.  Users who have a
compromised repository, and opam refuses to update to the cleaned up repository,
are asked to browse to the statement and validate its claim.  Once this is done,
they are guided to remove their local `.opam/repository` (and `.opam/keystate`)
and do a clean checkout.

## Keys and trust, revisited

There are two (and a half) kinds of keys, those of the snapshot bot, signed by
RMs, and those legible to sign packages they have a delegation for (or a quorum
of RMs).  An initial set of RMs is distributed with opam itself, upon revocation
or renewal a new vanity release of opam is done; but since only a quorum is
needed, revocation of a small subset of the distributed RM keys does not result
in opam refusing any repository.

Developers are still in full control (modulo housekeeping of RM) of their
packages: they can release new versions, publish new packages, add and remove
delegations (thus involve other developers).  If a developer looses their key,
a quorum of repository maintainers can establish a new key in the repository.

Repository managers are a dynamic group in the system, which grows and shrinks
over time.  These are people known and trusted enough in the community, that
it is unlikely that a quorum of RMs will collaborate on compromising the system.

Keys and packages are never deleted, but only marked for deletion. This serves
two purposes: a package name or identifier cannot be reused (and thus
potentially mixed with an old repository to claim ownership of packages), and
also there is some file to add signatures to (instead of using git metadata).

To prevent rollback attacks, we need to sprinkle `last-updated` fields.
Otherwise there is no way for the snapshot bot to check whether some
modification was just a replay of an earlier modified file.

Git, and GitHub is not used for security purposes.  Commits of new packages and
developer keys need to be signed by a RM; otherwise MITM between GitHub and
the snapshot bot might starve the name resource.  Key identifiers and package
names are case insensitive when compared for uniqueness.

Linearity checks can be skipped on the client, if the snapshot bot is trusted
(using `--skip-linearity`), which would be useful for low power devices. Maybe
introduction of a set of snapshot bots (and then a quorum again) is sensible?
(But this would need a slightly different structure, maybe dumping to
`snapshot/<id>` instead of a `snapshot` file would be enough, though.)

Attackers have to be evaluated at various levels: those stealing developer keys,
RM keys, snapshot keys; and then attacking either the entire repository, or a
single targeted user (or the snapshot bot).  When a developer key is stolen, the
snapshot key is also needed to target a single user for a malicious patch.

## Work and changes involved

### General

Write modules for key handling ; signing and verification of opam files.

Write the git synchronisation module with linearity checks.

### opam

Rewrite the default HTTP repository synchronisation module to use git fetch,
verify, and git pull. This should be fully transparent, except:
- in the cases of errors, of course
- when registering a non-official repository
- for some warnings with features that disable signatures, like source pinning
  (probably only the first time would be good)

Include the public root keys for the default repository, and implement
management of updated keys in `~/.opam/repo/name`.

Handle the new formats for checksums and non-repackaged archives.

Allow a per-repository security threshold (_e.g._ allow all, allow only signed
packages). It should be easy but explicit to add a local network, unsigned
repository. Backends other than git won't be signed anyway (local, rsync...).

### opam-publish

Generate keys, handle locally stored keys, generate `signature` files, handle
signing, submit signatures, check delegation, submit new delegation, request
delegation change (might require repository maintainer intervention if an RM
signed the delegation), delete developer, delete package.

Manage local keys. Probably including re-generating, and asking for revocation.

### opam-admin

Most operations on signatures and keys will be implemented as independent
modules (as to be usable from _e.g._ unikernels working on the repository). We
should also make them available from `opam-admin`, for testing and manual
management. Special tooling will also be needed by RMs.

- fetch the archives (but don't repackage as `pkg+opam.tar.gz` anymore)
- allow all useful operations for repository maintainers (maybe in a different
  tool ?):
  * manage their keys
  * list and sign changed packages directly
  * list and sign waiting delegations to developer keys
  * validate signatures, print reports
  * adding a signature to an existing file to meet the quorum
  * list quorums waiting to be met on a given branch
- generate `signed` tags (same as the snapshot bot, for testing)

### Signing bots

If we don't want to have this processed on the publicly visible host serving the
repository, we'll need a mechanism to fetch the repository, and submit the
`signed` tag back to the repository server.

Doing this through mirage unikernels would be cool, and provide good isolation.
We could imagine running this process regularly:

- fetch changes from the repository's git (GitHub)
- check for consistency (linearity)
- generate and sign the `signed` tag
- push tag back to the release repository

### Travis

All security information and check results should be available to RMs before
they make the decision to merge a commit to the repository. This means including
signature and linearity checks in a process running on Travis, or similarly on
every pull-request to the repository, and displaying the results in the GitHub
tracker.

This should avoid most cases where the snapshot bot fails the validation,
leaving it stuck (as well as any repository updates) until the bad commits are
rewritten.

## Some more detailed scenarios

### `opam init` and `update` scenario

On `init`, the client clones the repository and get and validate the `signed`
tag according to the current keyset (which must be rooted in the preserved
keys (either shipped with opam itself or on disk from a previous run). If all
goes well, the repository is used.

Then all files' signatures are checked following the trust chains, and copied to
the internal repository mirror opam will be using (`~/.opam/repo/<name>`). When
a package archive is needed, the download is done either from the repository, if
the file is mirrored, or from its upstream, in both cases with known file size
and upper limit: the download is stopped if going above the expected size, and
the file removed if it doesn't match both.

On subsequent updates, the process is the same except that a fetch operation is
done on the existing clone, and that the repository is forwarded to the new
`signed` tag only if linearity checks passed (and the update is aborted
otherwise).

### `opam-publish` scenario

* The first time a developer runs `opam-publish submit`, a developer key is
  generated, and stored locally.
* Upon `opam-publish submit`, the package is signed using the key, and the
  signature is included in the submission.
* If the key is known, and delegation for this package matches, all is good
* If the key is not already registered, it is added to `/keys` within the
  pull-request, self-signed.
* If there is no delegation for the package, the `/packages/pkgname/delegation`
  file is added, delegating to the developer key and signed by it.
* If there is an existing delegation that doesn't include the developer's key,
  this will require manual intervention from the repository managers. We may yet
  submit a pull-request adding the new key as delegate for this package, and ask
  the repository maintainers -- or former developers -- to sign it.

## Security analysis

We claim that the above measures give protection against:

- Arbitrary packages: an existing package cannot be updated unless it carries
  a valid signature of the original developer.  Compromising the developer key
  is sufficient to get this privilege (and once the snapshot bot signed the
  new malicious release, users will be able to install it).

- Rollback attacks: git updates must follow the currently known `signed` tag.
  If the snapshot bot detects deletions of packages, it refuses to sign, and
  clients double-check this.  All signed files contain a `last-updated` field.

- Indefinite freeze attacks: the snapshot bot periodically signs the `signed`
  tag with a timestamp, if a client receives a tag older than the expected age
  it will notice.

- Endless data attacks: we rely on the git protocol and this does not defend
  against endless data. Downloading of package archive (of which the origin may
  be any mirror), though, is protected. The scope of the attack is mitigated in
  our setting, because there are no unattended updates: the program is run
  manually, and interactively, so the user is most likely to notice.

- Slow retrieval attacks: same as above.

- Extraneous dependencies attacks: metadata is signed, and if the signature does
  not match, it is not accepted.

- Mix-and-match attacks: the repository has a linearity condition, and partial
  repositories are not possible.

- Malicious repository mirrors: if the signature does not match, reject.

- Wrong developer attack: if the developer is not in the delegation, reject.

### GitHub repository

Is the link between GitHub (opam-repository) and the signing bot special?
If there is a MITM on this link, no new arbitrary packages can be added (missing
RM signature). No existing package can be altered or deleted, otherwise
consistency condition above does not hold anymore and the signing bot will not
sign.

Certainly, the access can be frozen, thus the signing bot does not receive
updates, but continues to sign the old repository version.

### Snapshot key

If the snapshot key is compromised, an attacker is able to:

- Freeze, by forever re-signing the `signed` tag with an updated timestamp.

Most importantly, the attacker won't be able to tamper with existing packages.
This hugely reduces the potential of an attack, even with a compromised
snapshot key.

The attacks above would also require either a MITM between the repository and
the client, or a compromise of the opam repository: in the latter case, since
the linearity check is reproduces even from the clients:

- any tamper could be detected very quickly, and measures taken.
- a freeze would be detected as soon as a developer checks that their
  package is really online. That currently happens
  [several times a day][opam-repo-pulse].

The repository would then just have to be reset to before the attack, which git
makes as easy as it can get, revoking the compromised snapshot key and
introducing a new one.

In the time before the signing bot can be put back online with the new snapshot
key -- _i.e._ the breach has been found and fixed -- RMs could manually sign
timestamped `signed` tag before they expire (_e.g._ once a day) so as not to
hold back updates.

### Repository Maintainer keys

Repository maintainers are powerful, they can modify existing opam files and
sign them (as hot-fix), introduce new delegations for packages, etc.).

However, by requiring a quorum for sensitive operations, we limit the scope of a
single RM key compromise to the validation of new developer keys or delegations
(which should be the most common operation done by RMs): this enables to raise
the level of security of the new, malicious packages but otherwise doesn't
change much from what can be done with just access to the git repository.

A further compromise of a quorum of RM keys would allow to remove or tamper with
any developer key, delegation or package: any of these amounts to being able to
replace any package with a compromised version. Cleaning up would require
replacing all but the root keys, and resetting the repository to before any
malicious commit.

### Developer key

Add an arbitrary new release to the repository (or modify an existing).  If the
attacker sits between GitHub and snapshot bot, they can inject new packages,
or modify old releases, which will _not_ be noticed by the snapshot bot.

For attackers submitting a modification or new release via the GitHub PR
mechanism, the RM also observing the development of the package is likely to
notice and check out-of-band with the developers.

### Compromise of several keys

If an attacker succeeds to compromise the snapshot key and a RM key, they can
introduce an arbitrary amount of new packages, all in their custom universe.

If a quorum of RM keys are compromised, changes can be made to the repository.
Only if this is undetected after the snapshot bot signed those commits, clients
will be affected.

If a quorum of RM keys and the snapshot key are compromised, well, then we need
to start over and rethink the quorum or integrate some OpSec rating for whom
to hand out RM keys to.


## Difference to TUF

- we get away with fewer roles, simplify the system
- we use git (but not their signing)
- and already have a hash over the entire repository (for snapshot bot)
- TUF provides a mechanism for delegation, but it's both heavier and not
  expressive enough for what we wanted -- delegate to packages directly.
- We split in lots more files, and per-package ones, to fit with and nicely
  extend the git-based workflow that made the success of opam. The original TUF
  would have big json files signing for a lot of files, and likely to conflict.
  Both developers and repository maintainers should be able to safely work
  concurrently without issue. Signing bundles in TUF gives the additional
  guarantee that no file is removed without proper signature, but this is
  handled by git and signed tags.
- instead of a single file with all signed packages by a specific developer,
  one file per package

### Differences to Haskell:

- use TUF keys, not gpg
- e2e signing

[tuf]: http://theupdateframework.com/
[python-tuf]: http://www.python.org/dev/peps/pep-0458/
[ruby-tuf]: https://corner.squareup.com/2013/12/securing-rubygems-with-tuf-part-1.html
[haskell-tuf]: http://www.well-typed.com/blog/2015/04/improving-hackage-security/
[haskell-tuf-git]: https://github.com/commercialhaskell/commercialhaskell/wiki/Git-backed-Hackage-index-signing-and-distribution
[haskell-tuf-e2e]: https://github.com/commercialhaskell/commercialhaskell/wiki/Package-signing-detailed-propsal
[thandy]: http://google-opensource.blogspot.jp/2009/03/thandy-secure-update-for-tor.html
[nocrypto]: http://opam.ocaml.org/packages/nocrypto/nocrypto.0.3.1/
[tuf-tests]: https://github.com/theupdateframework/tuf/tree/develop/tests
[tuf-spec]: https://raw.githubusercontent.com/theupdateframework/tuf/develop/docs/tuf-spec.txt
[tuf-spec-priorities]: https://github.com/theupdateframework/tuf/blob/e034be2687fa4eacc6c05ffff8a0d8a387eb3d20/docs/tuf-spec.txt#L632
[opam-format]: https://opam.ocaml.org/doc/Manual.html#Generalfileformat
[opam-repo-pulse]: https://github.com/ocaml/opam-repository/pulse

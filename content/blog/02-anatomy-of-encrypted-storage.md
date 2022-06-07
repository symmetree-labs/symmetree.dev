+++
title = "Stash, Cache, and Keys: Anatomy of Encrypted Storage"
date = 2022-06-09
+++

If you have tuned into our [pilot episode](/blog/01-serious-backups),
you should have gotten a glimpse of the issues I see in the current
landscape of secure archiving tools, and *what* a new tool should
do. I, however, never elaborated on *how* that would happen. 

The [last release of
Zerostash](https://github.com/symmetree-labs/zerostash/releases/tag/v0.4.1)
now supports local caching of archives, S3 remotes, and natively using
macOS Keychain and keyfiles to store credentials. This provides great
apropo for me to elaborate on what makes this possible.

This is a long read, so buckle up, get a beverage, and put on your
monocle.

## Where We're Going

For those of you who have missed our previous programming (the British
programming, where they talk about the telly), Zerostash is solving
the following technical problems:

 * Store large amounts of files securely in random access archives.
 * Do not leak metadata like archive or file size.
 * Do not require extra server software apart from file storage.
 * The archives should be cheap to replicate, synchronize.
 * Store only incremental changes.
 * Stay as fast as possible for most use cases.

Ok, this (sorta) makes sense, but *"most use cases"* definitely sounds
like a thing to noodle on a bit more.

## Most Use Cases

Covering all use cases is hard, because everybody imagines doing
backups differently.

Some people will want to create a snapshot of their current work
environment, and occasionally send that off to their local or cloud
storage. Some will use hard drives and put them in a safe. Some will
use a combination of both. Some are afraid of Mossad, others are
afraid of [Not
Mossad](https://www.usenix.org/system/files/1401_08-12_mickens.pdf).

<div class="m-paragraph">
<div class="float-left w-50">
In addition, people are not only doing backups. Activists,
journalists, and researchers will want to preserve or, occasionally,
share stashes of documents with others securely without using
excessive bandwidth.
</div>

<div class="float-right w-25 p-10">
<img src="/02-anatomy-of-encrypted-storage/use-cases.svg">
</div>
</div>

To add extra complexity, all platforms now ship their own cloud
storage solutions built-in and ready to go. The common thing among
them is that neither of these services actually hide the data from the
storage provider. Yet, UX integration expectations from users are
high, the lines between local and remote are intentionally blurred by
operating system vendors.

It would be great if we could engineer a solution that does it all.

## Physical Layout

What we should be aiming for then, seems like a sort of overlay file
system that's versatile enough to be used across all operating systems.

To circumvent the latency implications of dealing with a lot of files,
stemming mostly from syscall overhead and round trip times in a
networked environment, we're going to need some packing.

Zerostash packs ChaCha20-Poly1305 encrypted chunks of data in 4MB
objects. These chunks use convergent encryption, which means the same
cleartext data will encrypt to the same cyphertext within the same
stash. Chunks can be part of a file, or some serialized data
structure that helps us index all the content in the stash.

<img class="centered w-50 m-paragraph" src="/02-anatomy-of-encrypted-storage/object-layout.svg">

Why 4MB? It's fast enough to transfer on most internet connections,
after all [many
websites](https://httparchive.org/reports/page-weight?lens=top1m&start=2017_08_01&end=latest&view=list)
are [larger than
that](https://idlewords.com/talks/website_obesity.htm).

Additionally, most modern SSDs will use a block size of [a few
megabytes](https://spdk.io/doc/ssd_internals.html), which means
handing round sized `write()` calls to the controller (through the
filesystem) will *theoretically* give us aligned access and better
performance.

If there isn't 4MiB worth of chunks in an object, Zerostash will fill
the rest with random, creating 4MiB blobs that look random from start
to finish.

Objects also have an ID, so we know how to call them. This ID serves
no real purpose, in the sense that it doesn't validate the integrity
of the object.

However, the ID needs to be some random value, because we want to be
able to rewrite *some* of the objects without changing their ID. Why
would we do that? Because specialized servers belong in Michelin-star
restaurants.

This object size and naming convention helps us synchronize partial
updates to the archive, regardless of the underlying file system,
syncing mechanism, or storage backend, while maintaining a relatively
decent throughput/latency tradeoff.

Detecting changes to an object is as simple as looking at the
modification time, and extra objects can be added or removed based on
comparing file listings.

Simple enough? Sure, let's see how to fill all this random with useful
stuff.

## Bootstrapping to Infinity

At the core of Zerostash is a library called
[Infinitree](https://docs.rs/infinitree/latest/infinitree/) to
handle... infinite trees.

In fact, Infinitree handles all the storage-y stuff I described above,
and allows Zerostash to use a high-level API for its model like this:

```rust
type ChunkIndex = fields::VersionedMap<Digest, ChunkPointer>;
type FileIndex = fields::VersionedMap<String, Entry>;

#[derive(Clone, Default, Index)]
pub struct Files {
    pub chunks: ChunkIndex,
    pub files: FileIndex,
}
```

So how, exactly, do we reach `ChunkIndex` and `FileIndex` from our
encrypted chunks sitting in randomly named objects?

Well, remember when I said random IDs serve no purpose? I lied. They
actually serve the purpose of hiding the root of the tree itself.
This is what happens when we open a tree:

<img class="centered w-75 m-paragraph" src="/02-anatomy-of-encrypted-storage/key-derivation.svg">

The name of the root object is derived from the user password, along
with the encryption keys. Specifically, the output of an Argon2 KDF is
used to derive [Blake3
subkeys](https://docs.rs/blake3/latest/blake3/fn.derive_key.html)
for [various purposes](https://github.com/symmetree-labs/infinitree/blob/b962de0775b68d69c53065bc22f09fb85612edc3/infinitree/src/crypto.rs#L79-L92).

The root object also has a special twist: it has a 512 byte header.
In Infinitree 0.8, the header is a simple `ChunkPointer` that looks like this:

```rust
pub(crate) struct RawChunkPointer {
    pub offs: u32,
    pub size: u32,
    pub file: ObjectId,
    pub hash: Digest,
    pub tag: Tag,
}
```

This root chunk pointer is contains all the information required for
us to get to the next chunk, which contains a list of chunk pointers
we call a `Stream`:

```rust
pub struct Stream(Vec<ChunkPointer>);
```

Reading through the chunks in the `Stream` will allow us to decrypt
the transaction list, which in turn allows us to select a version to
restore into memory (see the `VersionedMap`s in the code example
above!), through, you guessed it, more `Stream`s and `ChunkPointer`s.

The transaction list looks like this:

```rust
pub(crate) type Field = String;
pub(crate) type TransactionPointer = (CommitId, Field, Stream);

/// A list of transactions, represented in order, for versions and fields
pub(crate) type TransactionList = Vec<TransactionPointer>;
```

Going through the transactions belonging to a particular `Field` and
`CommitId`, we can deserialize the contents of our `ChunkIndex` and
`FileIndex` data structures mentioned above.

<img class="centered w-75 m-paragraph" src="/02-anatomy-of-encrypted-storage/header-to-transactions.svg">

We can roll up the transaction list from the `CommitId` which we want
to restore, up to the root, and filter the fields we're interested in.

For instance, there's no point restoring the full `ChunkIndex` if
we're only looking to run `0s checkout` on some files. We do need it,
however, when we want to stash more stuff in the stash with `0s commit`.

As you can see above, all `ChunkPointer`s include a `Tag`, which is
essentially the MAC of the cyphertext as generated by the
ChaCha20-Poly1305 cypher. The `CommitId` is then generated by taking
Blake3 hash of all the `ChunkPointer`s that are included in all
transactions that make up a commit as well as some additional
metadata, such as the previous commit's id, ensuring that all contents
of the index are sealed. This is very similar to how Git handles
commit hashes.

The transaction list is rewritten in its entirety on every commit to
save space on the storage. However, saving a new commit will always
create at least one new 4MiB object.

A minimum size for commits somewhat limits the utility of Zerostash
for large numbers of small changes, but *theoretically* there's
nothing stopping the implementation to track and re-use objects based
on utilization (ie. how much useless random do we put at the
end). Tracking where each object ends, then syncing and re-syncing
them will increase caching complexity, but potentially improves
privacy by creating less deterministic access patterns.

## Caching

Since the files that make up the stash mostly don't change, we can
cache them quite efficiently.

Better still, we will know if we need to invalidate parts of the
cache, because we can detect if the root object changed on the server
side, and whether it's newer than our local version.

If the root object changed on the server, we can simply re-sync the
objects that are referenced in the transaction list's `Stream`
descriptor based on their ETag, modification timestamp, or even hash
if our tools support it.

This also means that only a reasonably small portion of the stash
needs to be available to add more data into it: the index. The index
is typically less than 0.5% of the size of all data stored in the
stash, which means using a storage backend like Amazon Glacier is just
a matter of some medium-complexity code and a few tough decisions.

## Unfinished Business

The simple setup is, well, simple. There's plenty of room for
improvements, and hardening, though. For one, changing the master
password is not straightforward. It was also highlighted to me that
this design is also potentially susceptible to [partitioning oracle
attacks](https://eprint.iacr.org/2020/1491.pdf), although the risk
should be limited.

NixOS and macOS support are top of the priority right now, and
achieving a point where scheduled backups are both easy and manageable
should happen soon. Changing the master password, removing old
commits, and garbage collection will come straight after.

## Dirty Business

Going forward, I would love to continue working on Zerostash and the
surrounding Rust ecosystem in a focused way, and I am looking for
people who want to support this journey!

If you or your organization is in need of improving your data
management story, why not **hire me**?

Send me an email at **p at symmetree.dev**!

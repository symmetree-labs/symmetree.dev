+++
title = "Carrying API guarantees to the command line with Clap"
date = 2022-07-29
+++

One of the primary principles of modern cryptographic design is to be
resistant to misuse. In other words, a cryptographic primitive must
provide interfaces that cannot be used in a way that results in a
compromise of core security assurances.

There is a very good reason people in crypto worlds, both as in
-graphy and -currency, are enthusiastic about Rust: the type system is
rich enough that we can make invalid states inexpressible in the code.
This property allows a large number of assumptions to be validated at
compile time, therefore making our runtime reasoning about the states
in which the program *can* be much simpler.

This kind of well-defined world gets much more complex to control once
we try to expose the associated functionality on a CLI. In this fairly
long ride, I'll do a quick introduction of how it's done in one of the
core Rust cryptographic libraries that also underpins
[`rustls`](https://github.com/rustls/rustls), then dive into some
[Zerostash](https://github.com/symmetree-labs/zerostash) internals
that drove the [new command line
options](/blog/03-hardware-and-asymmetry-05/) in 0.5.

## Communicating intent in Ring

Take, a real life example from
[infinitree](https://docs.rs/infinitree/latest/infinitree/) of using
[ring](https://github.com/briansmith/ring), a Rust wrapper around
Google's [BoringSSL](https://boringssl.googlesource.com/boringssl/).

```rust
use ring::aead;

let key =
        aead::UnboundKey::new(&aead::CHACHA20_POLY1305, key.expose_secret()).expect("bad key");

let aead = aead::LessSafeKey::new(key);

let nonce = {
    let mut buf = Nonce::default();
    buf.copy_from_slice(&sealed[HEADER_CYPHERTEXT..]);
    aead::Nonce::assume_unique_for_key(buf)
};

let decrypted = aead
    .open_in_place(nonce, aead::Aad::empty(), &mut buf[..HEADER_CYPHERTEXT])
    .map_err(CryptoError::from)?;
```

There are a few things going on here, so let's go through what
happens, piece by piece, with an attention to how the API helps
understanding implementation.

```rust
aead::UnboundKey::new(&aead::CHACHA20_POLY1305, key.expose_secret()).expect("bad key");
```

First off,
[`UnboundKey`](https://docs.rs/ring/latest/ring/aead/struct.UnboundKey.html),
according to the docs, is a *"An AEAD key without a designated role or
nonce sequence."*. The `new()` constructor, as we're used to in Rust,
will create us a key that is *not bound* to any nonce sequence. We
can't, therefore accidentally use the wrong kind of key in an
unexpected place. It's a different type.

```rust
aead::LessSafeKey::new(key)
```

`LessSafeKey`? Huh? Less safe than *what*? Let's see what the
[~~fox~~](https://www.youtube.com/watch?v=jofNR_WkoCE) [docs](https://docs.rs/ring/latest/ring/aead/struct.LessSafeKey.html) say:

> Immutable keys for use in situations where `OpeningKey`/`SealingKey`
> and `NonceSequence` cannot reasonably be used.
>
> Prefer to use `OpeningKey`/`SealingKey` and NonceSequence when
> practical.


Ah, that makes sense! There are better ways of using the API, but they
are not always practical, so there's a *less safe version* that can be
used more freely!

```rust
aead::Nonce::assume_unique_for_key(buf)
```

The constructor's name itself is highlighting to the API user that the
[*n-once*](https://www.urbandictionary.com/define.php?term=Nonce)
value must be unique. It's an important implementation
detail, an externality that cannot be sufficiently safeguarded against
by the type system. Even if the `open_in_place` function consumes
`nonce`, there's no way for the type system to ensure it's globally
unique for every use of the `key` *in the universe*.

```rust
aead.open_in_place(nonce, aead::Aad::empty(), &mut buf[..HEADER_CYPHERTEXT])
```

And finally, a single glance at `open_in_place` will tell us
everything about the data and parameters of the actual decryption
operation.

## Controlling the API complexity

An API that programmers use requires very different UX considerations
from a tool on the command line. However, users of both will want most
of the same assurances.

While the underlying misuse-resistant API means it is easy to create
an opinionated system that does The Right Thing one way, introducing a
choice into this user experience is riddled with traps.

Before `infinitree` 0.9, there was no way to change the encryption
keys of a tree after creating it. This was a direct result of mostly
profiler-driven development, and I kind of left it there to fix it
later. Everything worked, and it was oh so simple!

For
[reasons](https://docs.rs/infinitree/0.9.0/infinitree/crypto/index.html#changing-keys),
changing keys requires some assumptions. Establishing the API that now
provides the right amount of flexibility, static type checking, and
ease of use, required a few iterations.

This is how one changes the password in `infinitree` 0.9, ignoring all
the `use`s:

<a name="the-code"></a>

```rust
let key = ChangeHeaderKey::swap_on_seal(
    UsernamePassword::with_credentials("username".to_string(),
                                       "old_password".to_string()).unwrap(),
    UsernamePassword::with_credentials("username".to_string(),
                                       "new_password".to_string()).unwrap(),
);

let mut tree = Infinitree::<VersionedMap<String, String>>::open(
    Directory::new("/storage").unwrap(),
    key
).unwrap();

tree.reseal();
```

Looks simple enough. It might be surprising at first that
there's a special `ChangeHeaderKey` type instead of just `swap_key`
for the `tree` instance.

This is because under the hood, I wanted to ensure that all
expressible key transitions are *safe*, and this is statically ensured
by the type system.

For what you see, you see, is a lie.

```rust
pub type UsernamePassword = KeyingScheme<Argon2UserPass, Symmetric>;
```

Internally, the `UsernamePassword` encryption scheme is a combination
of an encrypted header format, and symmetric AEAD cypher. Changing the
header format should be possible. Changing the internal symmetric keys
makes all data inaccessible.

`ChangeHeaderKey` uses generics enforce the rules.

```rust
pub struct ChangeHeaderKey<H, N, I> {
    opener: Arc<H>,
    sealer: Arc<N>,
    convergence: I,
}

impl<H, N, I> ChangeHeaderKey<H, N, I> {
    pub fn swap_on_seal(original: KeyingScheme<H, I>, new: KeyingScheme<N, I>) -> Self {
        Self {
            opener: original.header,
            sealer: new.header,
            convergence: original.convergence,
        }
    }
}
```

So far, this has scaled well enough for the current encryption schemes
in `infinitree`, and, since I am not planning on many new features
here, it will probably stick around for some time.

## Wiring up the command line with `clap`

First of all, if you've made it this far, congratulations. You deserve
a [break](https://youtu.be/X3uKmTdgmQg?t=136).

All this nonsense in `infinitree` is there to make
[Zerostash](https://github.com/symmetree-labs/zerostash/releases/tag/v0.5.0)
support some fancy modes of storage, that we need to expose on the
CLI.

Clap's declarative mode is amazing. Zerostash supports mostly the same
stuff in the TOML-based configuration language and the command line.

Compare and contrast.

```toml
[stash.s3.key]
source = "plaintext"
user = "backup@road-warrior"
password = "a very secure password"

[stash.s3.backend]
type = "s3"
bucket = "laptop-backup"
region = { name = "us-east-1" }
keys = ["access_key", "secret_key"]
```

```
0s commit --user backup@road-warrior s3://access_key:secret_key@us-east-1#/bucket/path /
```

Ok, you can't specify the password on the command line, but apart from
that, it's pretty much the same.

The trick is that both of the above examples are translated into a
symbolic representation of the configuration of a stash.

```rust
#[derive(Default, Clone, Debug, Deserialize, Serialize)]
pub struct Stash {
    pub key: Key,
    pub backend: Backend,
    pub alias: String,
}
```

And, it's symbolic all the way down, including the `Key` and `Backend` enums. To create a `Stash` instance, we're either directly deserializing it from TOML, or we leverage the `clap` arguments we can include anywhere that will generate a Stash instance for us.

```rust
#[derive(clap::Args, Clone, Debug)]
#[clap(group(
            ArgGroup::new("key")
                .args(&["keyfile", "keystring", "yubikey"]),
        ))]
pub struct StashArgs {
    pub stash: String,
    #[clap(flatten)]
    pub symmetric_key: SymmetricKey,
    #[clap(short, long, value_name = "PATH")]
    pub keyfile: Option<PathBuf>,
    #[clap(short = 'K', value_name = "TOML", long)]
    pub keystring: Option<String>,
    #[clap(short, long)]
    pub yubikey: bool,
}

```

All of this will help Zerostash figure out which keys to use, and how
to turn your command line into a symbolic `Stash`. All this stays
symbolic until Zerostash opens the `Stash` when everything is
evaluated and turned into `infinitree` types, and Zerostash can start
using the `Infinitree` database instance.

To change the key of a stash, we need to define the `change` subcommand:

```rust
#[derive(Command, Debug)]
pub struct Change {
    #[clap(flatten)]
    from: StashArgs,
    #[clap(subcommand)]
    cmd: ChangeCmd,
}
```

Through a bunch of new `clap::Args` annotations and getting gradually
more specific in how and why we want to change things, we get lost in
the details, such as `ChangeCmd`.

One big down side of the declarative `clap` code, is that for complex
interfaces, you'll have a type for *everything*. And in case you want
to re-use a subcommand *slightly* differently, you'll need to break
out different uses in different places to different structs.

I'm not saying this is pretty. This gets tedious, and you have to be
patient.

But to cut to the chase, eventually, we want to run [the
code](#the-code). Once we distill the command line options to usable
`Key` instances, eventually we need to go through the variants of the
`Key` `enum` to map everything into specific types. There's a helper
trait that allows the different supported keying schemes to turn into
a specific `infinitree::Key` instance.

```rust
pub trait KeyToSource {
    type Target;
    fn to_keysource(self, _stash_name: &str) -> Result<Self::Target>;
}
```

And then we end up executing the symbolic `Key` configuration to
create a `ChangeHeaderKey` instance, and elide the type. There are a
few helper macros to help reduce noise, which is considerable, but in
the end, it seems to be necessary to create a huge `match` block that
maps out the valid transitions.

```rust
macro_rules! change_key {
    ($stash:ident, $old:expr, $new:expr) => {
        Arc::new(infinitree::crypto::ChangeHeaderKey::swap_on_seal(
            $old.to_keysource($stash)?,
            $new.to_keysource($stash)?,
        ))
    };
}

impl KeyToSource for Key {
    type Target = infinitree::Key;

    fn to_keysource(self, stash: &str) -> Result<infinitree::Key> {
        Ok(match self {
		    ...

            Self::ChangeTo { old, new } => match (*old, *new) {
                (Key::Interactive, Key::Interactive) => {
                    change_key!(stash, old!(), new!())
                }
                (Key::Interactive, Key::Userpass(new)) => change_key!(stash, old!(), new),
				
                ...
                Map out all valid transitions...
                ...

                _ => bail!("Old and new keys are incompatible!"),
            },
        })
    }
}
```

This `infinitree::Key` instance can then be used to open and reseal a stash:

```rust
let mut stash = stash_cfg.try_open(Some(key)).expect("Stash cannot be opened");
if stash.reseal().is_err() {
    fatal_error("Failed to change key");
}
```

And the password has been changed!

## Conclusion

Thanks for sticking around for this long. My key takeaway is that
Rust helps enormously to control the complexity that arises when we
move from the string-heavy CLI interface to a strict API boundary.

None of it is magic, though, and there is a considerable amount of
work involved in mapping out a large suite of CLI functions,
considering all the externalities a CLI program has to consider, even
if it all condenses down to a fairly narrow API surface that in turn
hides the inner complexities of key management.

Although `clap`'s declarative style helps a lot keeping all the functionality
explicit, there is a considerable amount of type noise by making
all uses of the *similar but not quite the same* configurations mapped
into something specific.

The robustness of building around the explicit types might be worth
it, as most of the connections are explicitly checked, and chaining
the flow and dependencies is easy to track for the different functions
exposed by the CLI.

Happy hacking!

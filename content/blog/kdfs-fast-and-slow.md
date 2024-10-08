+++
title = "KDFs, Fast and Slow"
date = 2024-09-02
+++

# KDFs
I was just picking out a KDF (key derivation function) to use in a program I was writing the other day and I thought it would be interesting to explain what a KDF is, the different types of KDFs, and the differences in the functionality they provide. 

A KDF is a cryptographic algorithm used to derive one or more cryptographic keys from a given secret input. These secret inputs can be things like another secret key, a password, or a group element from a Diffie-Hellman key exchange. Because passwords / key exchange outputs aren't uniformly random (e.g. passwords are made up of words and characters, not just random bits) they aren't suitable for use in other cryptographic algorithms that expect a uniformly random secret directly (see this [Crypto StackExchange](https://crypto.stackexchange.com/questions/95716/difference-between-non-uniformly-random-and-uniformly-random) answer for a better explanation). This is where a KDF can help us.

KDFs have the nice property of taking nonuniformly random input of arbitrary length and creating a uniformly random output of arbitrary length. So we can send a password through a KDF and get an output which is suitable for use in other cryptographic algorithms. Pretty neat! Keep in mind that the entropy level of the input is still super important - running a poor password (e.g. the word "password") through a KDF will still produce a uniformly random output, but this output has low entropy. Garbage entropy in, garbage entropy out. Because a KDF is deterministic (which we'll see in some of the examples below), having sufficient entropy for the input is still important. 

Cool, now we know what a KDF is. How are they used in practice? Generally KDFs broadly fall into two categories:

- KDFs suitable for key derivation
- KDFs suitable for password storage

We'll go over what these two categories mean exactly in a moment, but first we should point out a major difference in behavior for these two use cases. Basically, KDFs used for key derivation are fast, and KDFs used for password storage are (purposefully) slow. Also note that you certainly can use a slow KDF for key derivation, but it will be slower and may not provide any additional benefit depending on your use case!

# Key Derivation
Taking a single secret key and deriving several more secret keys from it is a pretty common pattern in cryptography. Let's briefly look at how this is used in the Signal protocol.

## Signal KDF Chains
Signal uses what they call the [Double Ratchet algorithm](https://signal.org/docs/specifications/doubleratchet/) to encrypt their messages. Here's Signal's description of the algorithm:

> The Double Ratchet algorithm is used by two parties to exchange encrypted messages based on a shared secret key. Typically the parties will use some key agreement protocol (such as X3DH [1](https://signal.org/docs/specifications/doubleratchet/#ref-x3dh)) to agree on the shared secret key. Following this, the parties will use the Double Ratchet to send and receive encrypted messages.

In the Double Ratchet algorithm, KDFs are used to form a [KDF chain](https://signal.org/docs/specifications/doubleratchet/#kdf-chains). If you read their specification, you'll see the properties gained by using KDFs here (specifically, [HKDF](https://signal.org/docs/specifications/doubleratchet/#recommended-cryptographic-algorithms)). There are tons of keys being generated in the background when you use Signal - each message is encrypted with its own unique key. KDFs are what make this possible, given the same state on both sides of the conversation (the root key and the chain keys), both parties can derive the same key (because HKDF is deterministic) that makes it possible to encrypt the message on one end of the conversation and decrypt it on the other. Then the state is updated and new keys are derived as messages are sent and received. Signal calls this the symmetric-key ratchet, one of the two ratchets from the algorithm name. Obviously there is a lot more that goes into the Signal protocol which you can read at the links above. Their specification is surprisingly readable, even for a non-expert like myself!

## Hash function vs KDF
The follow-up question to the above example of using a KDF is why even use KDFs at all? Can we just use a standard hash function like SHA-512 to hash a secret key, then use that hash output as a key? Well, you can actually get away using a hash function for this sometimes! For the differences here, I'll defer to David Wong's excellent book, [Real-World Cryptography](https://www.manning.com/books/real-world-cryptography):

> HKDF is not the only way to derive multiple secrets from one secret. A more naive approach is to use hash functions. As hash functions do not expect a uniformly random input and produce uniformly random outputs, they are fit for the task. Hash functions are not perfect, though, as their interface does not take into account domain separation (no customization string argument) and their output length is fixed. Best practice is to avoid hash functions when you can use a KDF instead. Nonetheless, some well-accepted algorithms do use hash functions for this purpose. For example, you learned in chapter 7 about the Ed25519 signature scheme that hashes a 256-bit key with SHA-512 to produce two 256-bit keys.

So the rule of thumb here is to use KDFs over hash functions.

Great, we saw a good example of how deriving new cryptographic keys from other keys can be useful in a real world situation and how a KDF can differ from a standard cryptographic hash function. Let's look at the main use case for our "slow" KDFs: password storage.
# Password Storage
Hopefully we all know not to store passwords in cleartext, otherwise if they are compromised the attacker would know everyone's passwords! If you're unfamiliar with password storage, here's a brief overview: usually the password storage being compromised and then used is mostly mitigated by storing passwords in a hashed format. A client would send their password in cleartext (hopefully over a secure medium like TLS; the hash itself isn't sent because then it would [effectively become the password](https://en.wikipedia.org/wiki/Pass_the_hash)), the server would then hash the password and compare the hash (hopefully in constant time to avoid timing attacks) to what it has stored in its database. If the hash matches, the password is correct. In this situation, if an attacker gains access to all the password hashes, they must first crack the passwords by guessing password inputs, hashing them, and seeing if anything matches. Ideally we would be able to [prove knowledge of a password without sending it in cleartext](https://blog.cryptographyengineering.com/2018/10/19/lets-talk-about-pake/), but we are talking about KDFs here!

Anyways, the issue is that if you use a fast hash algorithm (like anything in the SHA family) to hash the passwords then store them, an attacker can perform many guesses very quickly to potentially recover the passwords. There's some ways to make this a bit more difficult for attackers, such as using a [salt](https://en.wikipedia.org/wiki/Salt_(cryptography)) or even a [pepper](https://en.wikipedia.org/wiki/Pepper_(cryptography)), but again we are talking about KDFs! There are KDFs specifically designed to be slow for this very purpose: to make brute force and dictionary attacks either impossible or very expensive. Some of these slow KDFs, like [Argon2](https://en.wikipedia.org/wiki/Argon2), are even [memory hard](https://en.wikipedia.org/wiki/Memory-hard_function), meaning they can only be optimized through speeding up memory access. This makes attacks against Argon2 more resistant to adversaries using specialized hardware like GPUs and ASICs, which are usually good at quickly hashing many possible password guesses but will be bottlenecked by the memory access requirement. As I mentioned earlier though, if you start with a low-entropy, easily-guessable password all of this protection will not do too much. These slow KDFs only slow things down - they don't really make passwords magically stronger. For a real world example of using Argon2, check out the open-source password manager Bitwarden, which [supports using Argon2](https://bitwarden.com/help/kdf-algorithms/#argon2id).

Awesome, now we have looked at the differences between fast KDFs and purposefully slow ones and know when they should be used.

# Summary
To summarize:
- KDFs can be used to derive suitable cryptographic keys from an unsuitable input (like a password).
- If the input is bad enough (low entropy), a KDF won't help much here.
- Fast KDFs (like HKDF) are good when you want to derive a key from another key. Technically a hash function can be used here sometimes, but better to be safe and just use a KDF.
- Slow KDFs (like Argon2) are good when you want to store a password. These KDFs are purposefully slow to increase the cost of brute-force and dictionary attacks.

Hope this was useful or at least interesting!

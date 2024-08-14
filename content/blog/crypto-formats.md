+++
title = "Understanding Cryptographic Formats: ASN.1, DER, PEM, and more!"
date = 2024-08-13
+++

Recently I was messing with implementing my own certificate validation during a TLS handshake. In my application the
certificates and their corresponding keypairs are generated on the fly, but before I got to that point I was just
testing with static certificates and keys I manually generated with OpenSSL. When writing the code to read these
and use them in my application I realized I didn't really know much about the specifics of the encoding formats used
for cryptographic keys, so I decided to dig in a bit.

# ASN.1
The first place to start is `ASN.1` (Abstract Syntax Notation One). `ASN.1` is just a language for defining data
structures so they can be serialized and deserialized in a standard way. If you've used something like Protocol Buffers
(aka `protobuf`), `ASN.1` is extremely similar in purpose. Here's a simple example:

```
Person ::= SEQUENCE {
    name UTF8String,
    age INTEGER
}
```

Here we have a simple `ASN.1` definition that defines a data structure called `Person` that has the fields `name` (of
type `UTF8String`) and `age` (of type `INTEGER`). `SEQUENCE` is just an ordered list of elements, like a struct in C.
Now here is an actual, albeit simple, `ASN.1` object as defined in [RFC 8017](https://datatracker.ietf.org/doc/html/rfc8017#autoid-43),
also known as `PKCS #1 v2.2`. We'll talk about `PKCS#1` more in a bit.

```
RSAPublicKey ::= SEQUENCE {
    modulus           INTEGER,  -- n
    publicExponent    INTEGER   -- e
}

The fields of type RSAPublicKey have the following meanings:
	- modulus is the RSA modulus n.
	- publicExponent is the RSA public exponent e.
```

This defines the data structure used to represent an RSA public key by the `PKCS#1` definition, and should be pretty
straightforward compared to our last example. `RSAPublicKey` is a `SEQUENCE` (think struct) that contains two fields,
both `INTEGER`s. Now we know how this structure is defined, but how is it actually encoded to be used in applications?

# DER
`ASN.1` has many different encoding schemes that specify how a specific data structure is actually represented in bytes.
For cryptography, we mostly only care about `DER` (Distinguished Encoding Rules) encoding. Some encoding formats of `ASN.1`
are very flexible in how a specific object can be encoded into bytes, `DER` is not. `DER` provides only one way to encode
an `ASN.1` value, which makes it ideal for use in cryptography where we don't want any ambiguity. This is why `DER` encoding
is extremely common for representing cryptographic entities like keys and certificates. `DER` is a
[type-length-value](https://en.wikipedia.org/wiki/Type%E2%80%93length%E2%80%93value) encoding. Because `DER` is a binary
encoding, you probably won't see it much when dealing with certificate and key files directly, this is where the more
human-friendly `PEM` encoding comes in.

# PEM
If you have seen a file that has a header looking something like `-----BEGIN CERTIFICATE-----`, you have seen `PEM`.
`PEM` is simply the base64 encoded `DER` bytes ([although not always](https://datatracker.ietf.org/doc/html/rfc7468#appendix-B))
with human readable headers and footers that give a hint as to what that data actually represents. `PEM` stands for
"Privacy-Enhanced Mail", this encoding was originally part of some standards that were never widely adopted in the email
space, but is now very commonly used to encode cryptographic materials.

Great, now we know that `ASN.1` is a language for defining how data structures are defined so they can be used in systems,
`DER` is the most commonly used encoding of that format in cryptography, and `PEM` is how the `DER` bytes are textually
encoded so humans don't have to deal with a binary format. Let's go back to `ASN.1` formats as there are several
commonly used in cryptography. We will be looking at the `DER` and `PEM` encodings of these structures and viewing
their decoding with the excellent [ASN.1 JavaScript Decoder](https://lapo.it/asn1js).

# PKCS#1
`PKCS#1` is a key format that is specific to RSA only. When looking at a public or private key that begins with
`-----BEGIN RSA PRIVATE KEY-----` or `-----BEGIN RSA PUBLIC KEY-----` you are working with a `PKCS1` formatted key. The
`ASN.1` formats can be found in [RFC8017](https://datatracker.ietf.org/doc/html/rfc8017#appendix-A), as we mentioned
before. OpenSSL calls this the "traditional" format. Here's an example of generating one in this format using OpenSSL 3.3:

```
~ % > openssl genrsa -traditional
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAjmkFvWuYRq87fDGbEyX+ePzE97eIYx+BuciKzXe9eSM7oL0P
I3vlDtXXiS2B5FOOmb97IMSqJ6mgjoS2CxEbD/SagNkjhMq3fyoiHU2kXSuw0Is9
...
2S3cTMnr/Mr3hAkgBvJMxanrIdyCsE42oM1y9Qwb0Xdybwq5i/ehyPqv+xSYzXO+
A+ZQg1Ng2CmUS7LaErp0gFGTnwsn2YmD92vMo79RIlj9c0FM43rW6Q==
-----END RSA PRIVATE KEY-----
```

And decoded it looks like:

```
RSAPrivateKey SEQUENCE (9 elem)
	- version Version INTEGER 0
	- modulus INTEGER (2048 bit) 1797762779...
	- publicExponent INTEGER 65537
	- privateExponent INTEGER (2044 bit) 1022340367...
	- prime1 INTEGER (1024 bit) 1401182203...
	- prime2 INTEGER (1024 bit) 1283032837...
	- exponent1 INTEGER (1024 bit) 9533135981...
	- exponent2 INTEGER (1020 bit) 9937401292...
	- coefficient INTEGER (1024 bit) 1108249090...
```

Which matches the definition in the RFC:

```
RSAPrivateKey ::= SEQUENCE {
    version           Version,
    modulus           INTEGER,  -- n
    publicExponent    INTEGER,  -- e
    privateExponent   INTEGER,  -- d
    prime1            INTEGER,  -- p
    prime2            INTEGER,  -- q
    exponent1         INTEGER,  -- d mod (p-1)
    exponent2         INTEGER,  -- d mod (q-1)
    coefficient       INTEGER,  -- (inverse of q) mod p
    otherPrimeInfos   OtherPrimeInfos OPTIONAL
}
```

If we don't specify we want the "traditional" format, `PKCS#8` is used.

# PKCS#8
`PKCS#8` is a more generalizable standard for storing private keys in that is isn't restricted only to RSA, which is why
it is preferred in more modern applications. The details and `ASN.1` structures can be found in
[RFC5208](https://datatracker.ietf.org/doc/html/rfc5208) and [RFC5958](https://datatracker.ietf.org/doc/html/rfc5958).
This is the default format when using OpenSSL and not specifying the "traditional" format. If you see the headers
`-----BEGIN PRIVATE KEY-----` or `----- BEGIN PUBLIC KEY -----`, you are dealing with `PKCS#8`.

```
~ % > openssl genpkey -algorithm ed25519
-----BEGIN PRIVATE KEY-----
MC4CAQAwBQYDK2VwBCIEICeijmCNKBgrpWFsec4l8vsUKb8rysrUM9Y8TqX+YeCu
-----END PRIVATE KEY-----
```

```
PrivateKeyInfo SEQUENCE (3 elem)
	- version Version INTEGER 0
	- privateKeyAlgorithm AlgorithmIdentifier SEQUENCE (1 elem)
	    - algorithm OBJECT IDENTIFIER 1.3.101.112 curveEd25519 (EdDSA 25519 signature algorithm)
	- privateKey PrivateKey OCTET STRING (34 byte) 042027A28E...
	    Offset: 12  
	    Length: 2+34  
	    (encapsulates)  
	    Value:  
	    **(34 byte)  
	    042027A28E...**
	    - OCTET STRING (32 byte) 27A28E608D...
```

The `PKCS#8` standard also defines optional password encryption for your private key. If you see
`----- BEGIN ENCRYPTED PRIVATE KEY -----`, that indicates an encrypted `PKCS#8` key.

```
~ % > openssl genpkey -algorithm ed25519 -aes256
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----BEGIN ENCRYPTED PRIVATE KEY-----
MIGjMF8GCSqGSIb3DQEFDTBSMDEGCSqGSIb3DQEFDDAkBBBNksGAfow4w6Kje5QV
g86/AgIIADAMBggqhkiG9w0CCQUAMB0GCWCGSAFlAwQBKgQQETJoNfit1sdfCm9H
Pzc4GQRAQgojnmSIsUyWSzlVLZB479/MNspEOobFionW64vkh6lqBdPFBq4fpnXe
fnuR7DlVG2Y+7NamDKjyBPdIJxhxAg==
-----END ENCRYPTED PRIVATE KEY-----
```

The encoded `DER` value contains information on how to decrypt the contained encrypted key. You can see that the size of
the `PEM`-encoded encrypted key is actually larger than its plaintext counterpart due to this extra information:

```
~ % > openssl genpkey -algorithm ed25519 | wc -c
     119
~ % > openssl genpkey -algorithm ed25519 -aes256 | wc -c
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
     302
```

# SEC1
`SEC1` formatted keys are specifically for elliptic curve cryptography. The `ASN.1` specifications can be found
[here](https://www.secg.org/sec1-v2.pdf) and in [RFC5915](https://datatracker.ietf.org/doc/html/rfc5915). The
header `-----BEGIN EC PRIVATE KEY-----` usually indicates that the key is in the `SEC1` format.

```
~ % > openssl ecparam -name prime256v1 -genkey -noout
-----BEGIN EC PRIVATE KEY-----
MHcCAQEEIM/Te/oqzVkBxVBdGYCp0kIsUO4ryvU9HNo2KVNODh6XoAoGCCqGSM49
AwEHoUQDQgAEkbxfrK8/qJic3VHVxgh3xmOhCGW/OxERLbC0utixI2q3lvQmNDS1
EOA9qTKFsxtMPGoYsgss21E0pzBfOZTomw==
-----END EC PRIVATE KEY-----
```

Funny enough, using the ASN.1 JavaScript Decoder seems to ambiguously parse this as a `PKIMessage` object defined in
[RFC4210 Section 5.1](https://datatracker.ietf.org/doc/html/rfc4210#section-5.1):

```
PKIMessage SEQUENCE (4 elem)
	- header PKIHeader [?] INTEGER 1
	- body PKIBody [?] OCTET STRING (32 byte) CFD37BFA2A...
	- protection [0] (1 elem)
	    - PKIProtection [?] OBJECT IDENTIFIER 1.2.840.10045.3.1.7 prime256v1 (ANSI X9.62 named elliptic curve)
	- extraCerts [1] (1 elem)
	    - SEQUENCE [?] BIT STRING (520 bit) 0000010010...
```

But in our case, this is incorrect as we should have an `ECPrivateKey` object as defined in the previously linked RFC5915.
This `ASN.1` definition looks like:

```
ECPrivateKey ::= SEQUENCE {
     version        INTEGER { ecPrivkeyVer1(1) } (ecPrivkeyVer1),
     privateKey     OCTET STRING,
     parameters [0] ECParameters {{ NamedCurve }} OPTIONAL,
     publicKey  [1] BIT STRING OPTIONAL
}
```

Which does, in fact, match our generated private key if we use a more primitive `ASN.1` parsing tool:

```
~ % > openssl ecparam -name prime256v1 -genkey -noout | openssl asn1parse
    0:d=0  hl=2 l= 119 cons: SEQUENCE
    2:d=1  hl=2 l=   1 prim: INTEGER           :01
    5:d=1  hl=2 l=  32 prim: OCTET STRING      [HEX DUMP]:18A01421F9...
   39:d=1  hl=2 l=  10 cons: cont [ 0 ]
   41:d=2  hl=2 l=   8 prim: OBJECT            :prime256v1
   51:d=1  hl=2 l=  68 cons: cont [ 1 ]
   53:d=2  hl=2 l=  66 prim: BIT STRING
```

We see we have a `SEQUENCE` that contains an `INTEGER` (matching `version`), an `OCTET STRING` (matching `privateKey`),
then the name of our curve (`prime256v1`) and an optional `publicKey`, which we do not have.

# Summary

To summarize everything:
- `ASN.1` is a language to define arbitrary data structures.
- `DER` is way to encode `ASN.1` structures into bytes in a non-ambiguous way.
- `PEM` is a way to encode bytes into a more human-friendly format that gives the user a hint as to what the data
actually is (e.g. a private key, certificate, etc)
- `PKCS#1` is an encoding for RSA keys. If you see `-----BEGIN RSA PRIVATE KEY-----`, this is probably a `PKCS#1` `DER`
and `PEM` encoded file.
- `PKCS#8` is a more generalizable encoding format for keys that supports many different types of keys. It also
supports encrypting the `PEM` contents with a password. If you see `-----BEGIN PRIVATE KEY-----` or `-----BEGIN ENCRYPTED PRIVATE KEY-----`, this is probably a `PKCS#8` `DER` and `PEM` encoded key.
- `SEC1` is a key format specifically for elliptic curve keys. If you see `-----BEGIN EC PRIVATE KEY-----`, this is
probably a `SEC1` `DER` and `PEM` encoded file.

Hope this helps to make some of these formats a bit more clear!

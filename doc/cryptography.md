# Crytographic specification of pwbox

pwbox implements a [RFC 8018][rfc8018]-like scheme for password-based encryption
and decryption. The scheme consists of two steps:
 1. Derive the key `DK` from the password, using `salt` as the cryptographic
    salt
 2. Use `DK` as the input to the symmetric encryption scheme to perform encryption
    or decryption
 
In other words, the encryption operation looks like
```none
salt = randomBytes(SALT_LENGTH)
DK = KDF(params; password, salt)
ciphertext = Enc(DK, plaintext)
```
and decryption is
```none
DK = KDF(params; password, salt)
plaintext = Dec(DK, ciphertext)
```

## Choice of crypto-primitives

pwbox uses:
  * **scryptsalsa208sha256** from the NaCl/libsodium fame as the key derivation
    function. scrypt is a well-studied KDF, [introduced in 2009][scrypt].
    scrypt is *memory-intensive*, i.e., the speed of its computation is limited
    by the RAM-CPU bandwidth and the RAM capacity rather than CPU speed alone.
    This makes scrypt harder to compute on specialized [ASICs][asic] and
    correspondingly increases the cost of the attacks on the scheme
  * **XSalsa20-Poly1305** authenticated encryption scheme, which consists
    of XSalsa20 stream cipher and Poly1305 message authentication code (MAC)
    
The salt length is 32 bytes according to the scryptsalsa208sha256
specification. The salt is generated using the `randomBytes` routine from
NaCl/libsodium.

scryptsalsa208sha256 consumes `opslimit` and `memlimit` integers as its parameters.
These parameters tune the amount of computations and RAM consumption, respectively.
pwbox allows to set these parameters, and uses the same defaults as scryptsalsa208sha256
for the interactive key derivation (`opslimit = 524288`, `memlimit = 16777216`).

As the `secretbox` cipher uses a 32-byte key and a 24-byte nonce, the KDF is configured
to generate 56-byte derived key `DK`. First 32 bytes of `DK` are used as the key
for secretbox, and the remaining 24 bytes are used as the nonce.

## Serialization

After encryption, the resulting ciphertext is combined with the salt and KDF parameters
to allow to decrypt it in the future based solely on the password.

The combined sequence is serialized into a byte array (`Uint8Array` in JavaScript):

|    Field   | Length, bytes | Comments | 
|------------|--------------:|----------|
| Algorithm ID | 8 | Equal to `'scrypt\0\0'`, with `\0` denoting zero bytes |
| `opslimit`   | 4 | Little-endian |
| `memlimit`   | 4 | Little-endian |
| Salt         | 32 |  |
| Secretbox    | >= 16 | Variable length, with 16 bytes allocated for MAC | 

Overall, there is a 64-byte overhead; quite large, but not critical for modern
architectures.

## Parameter validation

When creating or deserializing a pwbox, parameters are validated as follows:

<!--FIXME write about it when validation is implemented-->

[rfc8018]: https://tools.ietf.org/html/rfc8018
[scrypt]: http://www.tarsnap.com/scrypt/scrypt.pdf
[asic]: https://en.wikipedia.org/wiki/Application-specific_integrated_circuit
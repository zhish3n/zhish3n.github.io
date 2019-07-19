---
layout: post
title:  "Asymmetric Encryption with ECC Key Pairs"
date:   2019-06-13 18:19:25 +0800
permalink: "/ecc-asymmetric-encryption"
description: An example (in Java) of asymmetric encryption/decryption using ECC key pairs and the ECDH key agreement protocol. 
# permalink: /:categories/:year/:month
---

Asymmetric cryptography uses pairs of public-private keys to encrypt and decrypt information; private keys are kept secret, while public keys may be openly distributed. In a scenario where there is a Party A and a Party B, Party A can encrypt some information using its own private key and Party B's public key.

After Party A sends this encrypted information over to Party B, Party B can decrypt the information using its own private key and the public key of Party A. Importantly, during this exchange, only the encrypted information and the public keys of both parties are shared. Each party has no knowledge of either other's private keys (it is also kept private from the public), but due to the nature of the cryptographic algorithms used to generate such public-private key pairs, both parties are still able to obtain at the same unencrypted information.

A popular cryptosystem to use in asymmetric cryptography is RSA, which uses the easily-understood mathematical operation of factoring a product of two large primes. ECC (elliptic curve cryptography) is another approach fo asymmetric cryptography based on the algebraic structure of elliptic curves over finite fields. The primary benefit of ECC is a smaller key size while providing the same level of security as earlier asymmetric cyptosystems would. For encryption, a procedure called ECDH (elliptic curve Diffie-Hellman) is used with ECC key-pairs.

### Implementing ECC Encryption (in Java)

In this scenario, there are two parties – A and B. Both require a pair of ECC public-private keys. To generate an ECC key pair, use the code below.

```java
Security.addProvider(new BouncyCastleProvider());
ECGenParameterSpec spec = new ECGenParameterSpec("secp256r1");
KeyPairGenerator gen = KeyPairGenerator.getInstance("ECDH", "BC");
gen.initialize(spec, new SecureRandom());
KeyPair pair = gen.generateKeyPair();
ECPublicKey partyXPubKey = (ECPublicKey) pair.getPublic();
ECPrivateKey partyXPrivKey = (ECPrivateKey) pair.getPrivate();
```

Recall that party needs their own pair of keys so the code above should be run twice. The next step is for one of the parties to generate a session key from its key pair. Using this session key, they can then create a cipher to encrypt any message that they would like to send to the other party. This is shown below, from the perspective of a Party A.

```java
// 1. Generate the pre-master shared secret
KeyAgreement ka = KeyAgreement.getInstance("ECDH", "BC");
ka.init(partyAPrivKey);
ka.doPhase(partyBPubKey, true);
byte[] sharedSecret = ka.generateSecret();

// 2. (Optional) Hash the shared secret.
// 		Alternatively, you don't need to hash it.
MessageDigest messageDigest = MessageDigest.getInstance("SHA-256");
messageDigest.update(sharedSecret);
byte[] digest = messageDigest.digest();

// 3. (Optional) Split up hashed shared secret into an initialization vector and a session key
// 		Alternatively, you can just use the shared secret as the session key and not use an iv.
int digestLength = digest.length;
byte[] iv = Arrays.copyOfRange(digest, 0, (digestLength + 1)/2);
byte[] sessionKey = Arrays.copyOfRange(digest, (digestLength + 1)/2, digestLength);

// 4. Create a secret key from the session key and initialize a cipher with the secret key
SecretKey secretKey = new SecretKeySpec(sessionKey, 0, sessionKey.length, "AES");
Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding", "BC");
IvParameterSpec ivSpec = new IvParameterSpec(iv);
cipher.init(Cipher.ENCRYPT_MODE, secretKey, ivSpec);

// 5. Encrypt whatever message you want to send
String encryptMe = "Hello world!";
byte[] encryptMeBytes = encryptMe.getBytes(StandardCharsets.UTF_8);
byte[] cipherText = cipher.doFinal(encryptMeBytes);
String cipherString = Base64.getEncoder().encodeToString(cipherText);
```

> Recall that encrypting takes Party A's private key and Party B's public key.

> **Why do we use a session key?** If we were only concerned with a single party, then there would no point in using a session key. With a single party, we could just encrypt a message using the public key and then decrypt using the private key. In this case however, we have two parties. Neither party has knowledge of each other's private keys, so neither party can simply decrypt the message using the other party's private key.

Now that Party A has encrypted the message they want to send, Party B will try to decrypt it. The process for decrypting the message is very similar to the process of encrypting the message. Using its own private key and Party A's public key, Party B constructs the pre-master shared secret in the same way Party A did in the first step of the code block above. Note that both the pre-master shared secrets should be the exact same! Now that Party B has the exact same pre-master shared secret, they can hash the secret, split it up into an initialization vector and a session key, and initialize a cipher instance in the same way Party A did it in the code block above. The only difference is that in this case, the cipher should be initialized to `DECRYPT_MODE`, like shown below.

```java
// Same stuff as before...
cipher.init(Cipher.DECRYPT_MODE, secretKey, ivSpec);

// 5. Encrypt whatever message you want to send
String decryptMe = cipherString; // Message received from Party A
byte[] decryptMeBytes = Base64.getDecoder().decode(decryptMe);
byte[] textBytes = cipher.doFinal(decryptMeBytes);
String originalText = new String(textBytes);
```

The `originalText` String that Party B ends up with should be the same as the `encryptMe` String that Party A encrypted and sent over to Party B. In this whole exchange, both parties utilized the public keys of either party, but never shared their private key. Since the private keys are never shared and are essential in creating the pre-master shared secret which is used to encrypt/decrypt the message, no onlooker or outside party will be able to decrypt the message, even if they observed everything that was sent one party to another.

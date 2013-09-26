---
layout: post
title: "Cryptographic Message Syntax"
date: 2012-11-11 17:13
comments: true
categories: crypto
---

CMS is the [IETF](http://en.wikipedia.org/wiki/IETF)'s standardized approach to cryptographically secure messages.
It provides a BER encoded, ASN1 defined means of communicating the parameters and data of a message between recipients.

The most recent definitions come from [RFC 5652](http://tools.ietf.org/html/rfc5652) and it does do a good job of explaining how each operation works and what is required, it even provides some usage examples.
What is missing is a simple rundown of the different types of messages that goes into more detail than [Wikipedia](http://en.wikipedia.org/wiki/Cryptographic_Message_Syntax) but doesn't have you jumping straight into the RFC.

Data section should be thought of as a way of using a cryptographic function rather than a message all on its own.
Correlations will become obvious but EncryptedData is simply a way of portraying symmetrically encrypted data and AuthenticatedData is essentially a way of addressing and sending a MAC.
As with using normal crypto functions you will often need a combination to provide the required security and so CMS messages are designed to contain nested data sections.
A common example is a Signed data wrapping an Enveloped data to provide confidentiality and authenticity, or an Enveloped Digest message for confidentiality and integrity.

I realize this is by no means a comprehensive rundown but it should frame the situation and give an overview before you get to the RFC.
The meat of the post is a list of each CMS data type and the most important parts of the ASN1 definition as it should allow you to figure out how each segment is used and the chain of data you will want in your message.

### EncryptedData

The most simple CMS message just takes a symmetric key and encrypt some data.
How to give that key to someone else is outside this message.
It's defined as:

```
EncryptedData ::= SEQUENCE {
  version CMSVersion,
  encryptedContentInfo EncryptedContentInfo,
  unprotectedAttrs [1] IMPLICIT UnprotectedAttributes OPTIONAL }
```

### DigestedData

Provides a digest along with the plaintext.
Typically this is then wrapped by an EnvelopedData or such to provide Integrity to a message.

```
DigestedData ::= SEQUENCE {
  version CMSVersion,
  digestAlgorithm DigestAlgorithmIdentifier,
  encapContentInfo EncapsulatedContentInfo,
  digest Digest }
```

### SignedData

SignedData messages allow any number of certificates to sign a payload.
In the regular signing way, each signer creates a digest of the payload and encrypts it with their private key.
The message contains the certificate of the signers and can contain a store of certificates and CRLs for cert path validation by the receiver.

```
SignerInfo ::= SEQUENCE {
  version CMSVersion,
  sid SignerIdentifier,
  digestAlgorithm DigestAlgorithmIdentifier,
  signedAttrs [0] IMPLICIT SignedAttributes OPTIONAL,
  signatureAlgorithm SignatureAlgorithmIdentifier,
  signature SignatureValue,
  unsignedAttrs [1] IMPLICIT UnsignedAttributes OPTIONAL }


SignedData ::= SEQUENCE {
  version CMSVersion,
  digestAlgorithms DigestAlgorithmIdentifiers,
  encapContentInfo EncapsulatedContentInfo,
  certificates [0] IMPLICIT CertificateSet OPTIONAL,
  crls [1] IMPLICIT RevocationInfoChoices OPTIONAL,
  signerInfos SignerInfos }
```

### EnvelopedData

EnvelopedData allows you to address encrypted data to any number of specific recipients.
The most common recipients are a certificate (or the private key of), a symmetric key or a password.
The payload is encrypted with a symmetric key and then this key is encrypted with the key provided by the recipient (or PBKDF for passwords).
So on decoding the recipient will decrypt the symmetric key from the RecipientInfo associated with them and then use that to decrypt the plaintext mesage.

```
EnvelopedData ::= SEQUENCE {
  version CMSVersion,
  originatorInfo [0] IMPLICIT OriginatorInfo OPTIONAL,
  recipientInfos RecipientInfos,
  encryptedContentInfo EncryptedContentInfo,
  unprotectedAttrs [1] IMPLICIT UnprotectedAttributes OPTIONAL }

RecipientInfo ::= CHOICE {
  ktri KeyTransRecipientInfo,
  kari [1] KeyAgreeRecipientInfo,
  kekri [2] KEKRecipientInfo,
  pwri [3] PasswordRecipientinfo,
  ori [4] OtherRecipientInfo }
```

### AuthenticatedData

AuthenticatedData is the one that always trips me up, it allows you to send a message that is only verifiable by a number of specific recipients.
It generates a new MAC secret key and with it generates a MAC digest for the plaintext.
It then encrypts the MAC secret key to any number of recipients similar to EnvelopedData and includes the MAC in the message.
The message itself is not encrypted and can be retrieved, but you cannot assert the message integrity without being one of the explicit recipients.

```
AuthenticatedData ::= SEQUENCE {
  version CMSVersion,
  originatorInfo [0] IMPLICIT OriginatorInfo OPTIONAL,
  recipientInfos RecipientInfos,
  macAlgorithm MessageAuthenticationCodeAlgorithm,
  digestAlgorithm [1] DigestAlgorithmIdentifier OPTIONAL,
  encapContentInfo EncapsulatedContentInfo,
  authAttrs [2] IMPLICIT AuthAttributes OPTIONAL,
  mac MessageAuthenticationCode,
  unauthAttrs [3] IMPLICIT UnauthAttributes OPTIONAL }
```

---
title: Compact TLS 1.3
abbrev: cTLS 1.3
docname: draft-ietf-tls-ctls-latest
category: info

ipr: trust200902
area: General
workgroup: TLS Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: E. Rescorla
    name: Eric Rescorla
    organization: Mozilla
    email: ekr@rtfm.com

 -
    ins: R. Barnes
    name: Richard Barnes
    organization: Cisco
    email: rlb@ipv.sx

 -
    ins: H. Tschofenig
    name: Hannes Tschofenig
    organization: Arm Limited
    email: hannes.tschofenig@arm.com

normative:
  RFC2119:

informative:



--- abstract

This document specifies a "compact" version of TLS 1.3. It is
isomorphic to TLS 1.3 but saves space by trimming obsolete material,
tighter encoding, and a template-based specialization technique. cTLS
is not directly interoperable with TLS 1.3, but it should eventually be
possible for a cTLS/TLS 1.3 server to exist and successfully interoperate.


--- middle

# Introduction

DISCLAIMER: This is a work-in-progress draft of cTLS and has not yet
seen significant security analysis, so could contain major errors. It
should not be used as a basis for building production systems.

This document specifies a "compact" version of TLS 1.3 {{!RFC8446}}. It is isomorphic
to TLS 1.3 but designed to take up minimal bandwidth. The space reduction
is achieved by four basic techniques:

- Omitting unnecessary values that are a holdover from previous versions
  of TLS.
- Omitting the fields and handshake messages required for preserving backwards-compatibility
  with earlier TLS versions.
- More compact encodings.
- A template-based specialization mechanism that allows pre-populating information 
  at both endpoints without the need for negotiation. 

For the common (EC)DHE handshake with pre-established certificates, cTLS
achieves an overhead of 45 bytes over the minimum required by the
cryptovariables.  For a PSK handshake, the overhead is 21 bytes.  Annotated
handshake transcripts for these cases can be found in {{transcripts}}.

Because cTLS is semantically equivalent to TLS, it can be viewed either
as a related protocol or as a compression mechanism. Specifically, it
can be implemented by a layer between the TLS handshake state
machine and the record layer.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

Structure definitions listed below override TLS 1.3 definitions; any PDU
not internally defined is taken from TLS 1.3 except for replacing integers
with varints.

# Common Primitives

## Varints

cTLS makes use of variable-length integers in order to allow a wide
integer range while still providing for a minimal encoding. The
width of the integer is encoded in the first two bits of the field
as follows, with xs indicating bits that form part of the integer.


| Bit pattern                | Length (bytes) |
|:----------------------------|:----------------|
| 0xxxxxxx                   | 1              |
|                            |                |
| 10xxxxxx xxxxxxxx          | 2              |
|                            |                |
| 11xxxxxx xxxxxxxx xxxxxxxx | 3              |

Thus, one byte can be used to carry values up to 127.

In the TLS syntax variable integers are denoted as "varint" and
a vector with a top range of a varint is denoted as:

~~~~~
     opaque foo<1..V>;
~~~~~

cTLS replaces all integers in TLS with varints, including:

* Values of uint8, uint16, uint24, uint32, and uint64
* Vector length prefixes
* Enum / code point values

We do not show the structures which only change in this way.

This allows implementations' encoding and decoding logic to implement cTLS
simply by having a mode in which integers always use the varint encoding.  Note
that if implementations treat opaque data in the same way as `uint8` values,
they MUST NOT convert the bytes of an opaque value to varints.

As an example, suppose we are given the following struct:

~~~~
      struct {
          uint32 FieldA;
          opaque FieldB<0..2^16-1>;
      } ExampleStruct;
~~~~

Encoding a value of this type with values FieldA=0x0A and FieldB=0x0B0B0B0B0B
would result in the following octet strings in "normal" (RFC 8446) and "compact"
modes, respectively:

~~~~~
Normal:  0000000A00050B0B0B0B0B
Compact: 0A050B0B0B0B0B
~~~~~


## Template-based Specialization

The transmission overhead in TLS 1.3 is largely contributed to by two factors, 
namely 
- by the negotiation of algorithm parameters, and extensions,  as well as 
- by the exchange of certificates. 

TLS 1.3 supports different credential types and modes that 
are impacted differently by a compression scheme. For example, TLS supports 
certificate-based authentication, raw public key-based authentication as well 
as pre-shared key (PSK)-based authentication. PSK-based authentication can be 
used with externally configured PSKs or with PSKs established through tickets. 

The basic idea of template-based specialization is that we start with the basic 
TLS 1.3 handshake, which is fully general and then remove degrees of freedom, 
eliding parts of the handshake which are used to express those degrees of
freedom. For example, if we only support one version of TLS, then it
is not necessary to have version negotiation and the
supported_versions extension can be omitted.

Importantly, this process is performed only for the wire encoding but
not for the handshake transcript.  The result is that the transcript for a
specialized cTLS handshake is the same as the transcript for a TLS 1.3 handshake
with the same features used.

One way of thinking of this is as if specialization is a stateful compression
layer between the handshake and the record layer:

~~~~~
+---------------+---------------+---------------+
|   Handshake   |  Application  |     Alert     |
+---------------+---------------+---------------+    +---------+
|               cTLS Compression Layer          |<---| Profile |
+---------------+---------------+---------------+    +---------+
|          cTLS Record Layer / Application      |
+---------------+---------------+---------------+
~~~~~

By assuming that out-of-band agreements took place already prior to the start of
the cTLS protocol exchange, the amount of data exchanged can be radically reduced. 
Because different clients may use different compression templates and because multiple 
compression templates may be available for use in different deployment environments, 
a client needs to inform the server about the profile it is planning to use. The 
profile field in the ClientHello serves this purpose. 

While it is possible to apply the template-based serialization to the entire 
TLS 1.3 protocol, this version of the specification focuses on the certificate-based
exchange. For other exchanges, the standard TLS 1.3 protocol is used instead. Most
exchanges in TLS 1.3 are highly optimized and do not require compression to be used. 

The compression profile defining the use of algorithms, algorithm parameters, and 
extensions is specified via a JSON dictionary.

For example, the following specialization describes a protocol
with a single fixed version (TLS 1.3) and a single fixed
cipher suite (TLS_AES_128_GCM_SHA256). On the wire, ClientHello.cipher_suites,
ServerHello.cipher_suites, and the supported_versions extensions in the
ClientHello and ServerHello would be omitted.

~~~~
{
   "version" : 772,
   "cipherSuite" : "TLS_AES_128_GCM_SHA256"
}
~~~~

The following elements are defined: 

profile (integer): 
: identifies the profile being defined. 

version (integer):
: indicates that both sides agree to the
single TLS version specified by the given integer value
(772 == 0x0304 for TLS 1.3). The supported_versions extension
is omitted from ClientHello.extensions and reconstructed in the transcript as a
single-valued list with the specified value. The supported_versions extension is
omitted from ClientHello.extensions and reconstructed in the transcript with the
specified value.

cipherSuite (string):
: indicates that both sides agree to
the single named cipher suite, using the "TLS_AEAD_HASH" syntax
defined in {{RFC8446}}, Section 8.4. The ClientHello.cipher_suites
field is omitted and reconstructed in the transcript as a single-valued
list with the specified value. The server_hello.cipher_suite field is
omitted and reconstructed in the transcript as the specified value.

dhGroup (string):
: specifies a single DH group to use for key establishment. The
group is listed by the code point name in {{RFC8446}}, Section 4.2.7.
(e.g., x25519). This implies a literal "supported_groups" extension
consisting solely of this group.

signatureAlgorithm (string):
: specifies a single signature scheme to use for authentication. The
group is listed by the code point name in {{RFC8446}}, Section 4.2.7.
(e.g., ed25519). This implies a literal "signature_algorithms" extension
consisting solely of this group.

random (integer):
: indicates that the ClientHello.Random and ServerHello.Random values
are truncated to the given values. When the transcript is
reconstructed, the Random is padded to the right with 0s and the
anti-downgrade mechanism in {{RFC8446)}, Section 4.1.3 is disabled.
IMPORTANT: Using short Random values can lead to potential
attacks. The Random length MUST be less than or equal to 32 bytes.
When this element is omitted, the random values are created by hashing 
the ephemeral public key via the hash algorithm indicated in 
cipherSuite and truncated to 32 bytes. 

mutualAuth (boolean):
: indicates whether the client has to be prepared to transmit a 
Certificate and a CertificateVerify message and whether the server 
has to transmit a CertificateRequest message. 

extension_order: 
: indicates in what order extensions appear in respective messages. 
This allows to omit sending the type. If there is only a single 
extension to be transmitted, then the extension length field can also
be omitted. For example, imagine that only the KeyShare extension 
needs to be sent in the ClientHello as the only extension. Then, 
the following structure

~~~
   28                    // Extensions.length
   33 26                 // KeyShare
     0024                // client_shares.length
       001d              // KeyShareEntry.group
       0020 a690...af948 // KeyShareEntry.key_exchange
~~~       

is compressed down to

~~~
   0020 a690...af948 // KeyShareEntry.key_exchange
~~~

clientHelloExtensions (predefined extensions):
: Predefined ClientHello extensions, see {predefined-extensions}

serverHelloExtensions (predefined extensions):
: Predefined ServerHello extensions, see {predefined-extensions}

encryptedExtensions (predefined extensions):
: Predefined EncryptedExtensions extensions, see {predefined-extensions}

certRequestExtensions (predefined extensions):
: Predefined CertificateRequest extensions, see {predefined-extensions}

knownCertificates (known certificates):
: A compression dictionary for the Certificate message, see {known-certs}

finishedSize (integer):
: indicates that the Finished value is to be truncated to the given
length. When the transcript is reconstructed, the remainder of the
Finished value is filled in by the receiving side.
[[OPEN ISSUE: How short should we allow this to be? TLS 1.3 uses
the native hash and TLS 1.2 used 12 bytes. More analysis is needed
to know the minimum safe Finished size. See {{RFC8446}};
Section E.1 for more on this, as well as
https://mailarchive.ietf.org/arch/msg/tls/TugB5ddJu3nYg7chcyeIyUqWSbA.]]

### Requirements on the TLS Implementation

To be compatible with the specializations described in this section, a
TLS stack needs to provide two key features:

If specialization of extensions is to be used, then the TLS stack MUST order
each vector of Extension values in ascending order according to the
ExtensionType.  This allows for a deterministic reconstruction of the extension
list.

If truncated Random values are to be used, then the TLS stack MUST be
configurable to set the remaining bytes of the random values to zero.  This
ensures that the reconstructed, padded random value matches the original.

If truncated Finished values are to be used, then the TLS stack MUST be
configurable so that only the provided bytes of the Finished are verified,
or so that the expected remaining values can be computed.

### Predefined Extensions

Extensions used in the ClientHello, ServerHello, EncryptedExtensions, and
CertificateRequest messages can be "predefined" in a compression profile, so
that they do not have to be sent on the wire.  A predefined extensions object is
a dictionary whose keys are extension names specified in the TLS
ExtensionTypeRegistry specified in {{RFC8446}}.  The corresponding value is a
hex-encoded value for the ExtensionData field of the extension.

When compressing a handshake message, the sender compares the extensions in the
message being compressed to the predefined extensions object, applying the
following rules:

* If the extensions list in the message is not sorted in ascending order by
  extension type, it is an error, because the decompressed message will not
  match.
* If there is no entry in the predefined extensions object for the type of the
  extension, then the extension is included in the compressed message
* If there is an entry:
    * If the ExtensionData of the extension does not match the value in the
      dictionary, it is an error, because decompression will not produce the
      correct result.
    * If the ExtensionData matches, then the extension is removed, and not
      included in the compressed message.

When decompressing a handshake message the receiver reconstitutes the original
extensions list using the predefined extensions:

* If there is an extension in the compressed message with a type that exists in
  the predefined extensions object, it is an error, because such an extension
  would not have been sent by a sender with a compatible compression profile.
* For each entry in the predefined extensions dictionary, an extension is added
  to the decompressed message with the specified type and value.
* The resulting vector of extensions MUST be sorted in ascending order by
  extension type.

Note that the "version", "dhGroup", and "signatureAlgorithm" fields in the
compression profile are specific instances of this algorithm for the
corresponding extensions.

[[OPEN ISSUE: Are there other extensions that would benefit from special
treatment, as opposed to hex values.]]

### Known Certificates {#known-certs}

Certificates are a major contributor to the size of a TLS handshake.  In order
to avoid this overhead when the parties to a handshake have already exchanged
certificates, a compression profile can specify a dictionary of "known
certificates" that effectively acts as a compression dictionary on certificates.

A known certificates object is a JSON dictionary whose keys are strings containing
hex-encoded compressed values.  The corresponding values are hex-encoded strings
representing the uncompressed values.  For example:

~~~~~
{
  "00": "3082...",
  "01": "3082...",
}
~~~~~

When compressing a Certificate message, the sender examines the cert_data field
of each CertificateEntry.  If the cert_data matches a value in the known
certificates object, then the sender replaces the cert_data with the
corresponding key.  Decompression works the opposite way, replacing keys with
values.

Note that in this scheme, there is no signaling on the wire for whether a given
cert_data value is compressed or uncompressed.  Known certificates objects
SHOULD be constructed in such a way as to avoid a uncompressed object being
mistaken for compressed one and erroneously decompressed.  For X.509, it is
sufficient for the first byte of the compressed value (key) to have a value
other than 0x30, since every X.509 certificate starts with this byte.

## Record Layer

The cTLS Record Layer assumes that records are externally framed
(i.e., that the length is already known because it is carried in a UDP
datagram or the like). Depending on how this was carried, you might
need another byte or two for that framing. Thus, only the type byte
need be carried and TLSPlaintext becomes:

~~~~
      struct {
          ContentType type;
          opaque fragment[TLSPlaintext.length];
      } TLSPlaintext;
~~~~

In addition, because the epoch is known in advance, the
dummy content type is not needed for the ciphertext, so
TLSCiphertext becomes:

~~~~
      struct {
          opaque content[TLSPlaintext.length];
          ContentType type;
          uint8 zeros[length_of_padding];
      } TLSInnerPlaintext;

      struct {
          opaque encrypted_record[TLSCiphertext.length];
      } TLSCiphertext;
~~~~

Note: The user is responsible for ensuring that the sequence
numbers/nonces are handled in the usual fashion.


## Handshake Layer

The cTLS handshake framing is same as the TLS 1.3 handshake
framing, except for two changes:

1. The length field is omitted
1. The HelloRetryRequest message is a true handshake message
   instead of a specialization of ServerHello.

~~~~
      struct {
          HandshakeType msg_type;    /* handshake type */
          select (Handshake.msg_type) {
              case client_hello:          ClientHello;
              case server_hello:          ServerHello;
              case hello_retry_request:   HelloRetryRequest;
              case end_of_early_data:     EndOfEarlyData;
              case encrypted_extensions:  EncryptedExtensions;
              case certificate_request:   CertificateRequest;
              case certificate:           Certificate;
              case certificate_verify:    CertificateVerify;
              case finished:              Finished;
              case new_session_ticket:    NewSessionTicket;
              case key_update:            KeyUpdate;
          };
      } Handshake;
~~~~

Empty messages are not transmitted. For example, if an 
EncryptedExtensions message contains no extension then the 
entire message is omitted. 

# Handshake Messages

In general, we retain the basic structure of each individual
TLS handshake message. However, the following handshake
messages have been modified for space reduction and cleaned
up to remove pre TLS 1.3 baggage.

## ClientHello

The cTLS ClientHello is as follows. The random value 
may be omitted as well as the cipher_suite. Extensions 
may also be omitted. 

~~~~
      opaque Random[RandomLength]; 

      struct {
          varint profile; 
          Random random;
          CipherSuite cipher_suites<1..V>;
          Extension extensions<1..V>;
      } ClientHello;
~~~~

## ServerHello

We redefine ServerHello in a similar way:
The random value may be omitted as well as the cipher_suite.
Extensions may also be omitted. 

~~~~
      struct {
          Random random;
          CipherSuite cipher_suite;
          Extension extensions<1..V>;
      } ServerHello;
~~~~

## HelloRetryRequest

The HelloRetryRequest has the following format:

~~~~
      struct {
          CipherSuite cipher_suite;
          Extension extensions<2..V>;
      } HelloRetryRequest;
~~~~

It is the same as the ServerHello above but without the unnecessary
sentinel Random value.

## EncryptedExtensions 

The EncryptedExtensions message is used to carry 
extensions that have prior to TLS 1.3 been conveyed in the 
ServerHello. If there are no extensions to be transmitted 
with the EncryptedExtensions message then this message is 
omitted. 

~~~~
      struct {
          Extension extensions<0..2^16-1>;
      } EncryptedExtensions;
~~~~
      
      
## CertificateRequest

If the setting of the certificate_request_context parameter in the 
certificateRequestExtensions field of the template compression profile 
is set to zero then the certificate_request_context is omitted. 

If there no extensions then the extension field is omitted. 

~~~~
      struct {
          opaque certificate_request_context<0..2^8-1>;
          Extension extensions<2..2^16-1>;
      } CertificateRequest;
~~~~

If this message is empty, then the entire message is omitted. 

## Certificate 

The configuration of the certificate_request_context parameter in the 
certificateRequestExtensions field of the template compression profile 
determines whether the certificate_request_context field can be omitted 
in the Certificate message. A value of zero in the certificate_request_context
parameter indicates the desire to omit this field.  
    
The extensions field is omitted when no extensions are present. 

~~~~
      enum {
          X509(0),
          RawPublicKey(2),
          cert_id(TBD1),
          (255)
      } CertificateType;

      struct {
          select (certificate_type) {
              case RawPublicKey:
                /* From RFC 7250 ASN.1_subjectPublicKeyInfo */
                opaque ASN1_subjectPublicKeyInfo<1..2^24-1>;

              case X509:
                opaque cert_data<1..2^24-1>;
                
              case cert_id: 
                opaque key_id<1..^8>;
          };
          Extension extensions<0..2^16-1>;
      } CertificateEntry;

      struct {
          opaque certificate_request_context<0..2^8-1>;
          CertificateEntry certificate_list<0..2^24-1>;
      } Certificate;
~~~~
      
## CertificateVerify

The CertificateVerify message does not contain the signature algorithm 
because it can be obtained from the signatureAlgorithm parameter of the
template compression profile. 

~~~~
      struct {
          opaque signature<0..2^16-1>;
      } CertificateVerify;
~~~~
      
## Finished

The Finished message contains no length information because the length is 
determined by the finishedSize parameter of the template compression profile. 

~~~~
      struct {
          opaque verify_data[finishedSize];
      } Finished;
~~~~

# Examples

The following section provides some example specializations.

TLS 1.3 only:

~~~~
{
   "Version" : 0x0304
}
~~~~

TLS 1.3 with AES_GCM and X25519 and ALPN h2, short random values,
and everything else is ordinary TLS 1.3.

~~~~
{
   "Profile" : 1, 
   "Version" : 772,
   "Random": 16,
   "CipherSuite" : "TLS_AES_128_GCM_SHA256",
   "DHGroup": "X25519",
   "Extensions": {
      "named_groups": 29,
      "application_layer_protocol_negotiation" : "030016832",
      "..." : null
    }
}
~~~~

Version 772 corresponds to the hex representation 0x0304, named group "29"
(0x001D) represents X25519.

[[OPEN ISSUE: Should we have a registry of well-known profiles?]]

# Security Considerations

WARNING: This document is effectively brand new and has seen no
analysis. The idea here is that cTLS is isomorphic to TLS 1.3, and
therefore should provide equivalent security guarantees.

The use of key ids is a new feature introduced in this document, which
requires some analysis, especially as it looks like a potential source
of identity misbinding. This is, however, entirely separable
from the rest of the specification.

The idea of hashing the ephemeral public keys and their use as random 
values was introduced by Karthik Bhargavan and requires analysis. 

Transcript expansion also needs some analysis and we need to determine
whether we need an extension to indicate that cTLS is in use and with
which profile.

# IANA Considerations

This document has no IANA actions.

--- back

# Example Exchange {#transcripts}

The follow exchange illustrates a complete cTLS-based exchange supporting 
mutual authentication using certificates. The digital signatures use ECDSA with SHA256 
and NIST P256r1. The ephemeral Diffie-Hellman uses the FX25519 curve and 
the exchange negotiates TLS-AES-128-CCM8-SHA256. 
The certificates are exchanged using certificate identifiers. 

The resulting byte counts are as follows:

~~~~~
                     ECDHE
              ------------------
              TLS  CTLS  Overhead
              ---  ----  --------
ClientHello   132   36       4
ServerHello    90   36       4
ServerFlight  478   80       7
ClientFlight  458   80       7
==================================
Total        1158  232      22
~~~~~


The following compression profile was used in this example:

~~~~~
{
  "profile": 1, 
  "version": 772,
  "cipherSuite": "TLS_AES_128_CCM_8_SHA256",
  "dhGroup": "X25519",
  "signatureAlgorithm": "ECDSA_P256_SHA256",
  "finishedSize": 8,
  "clientHelloExtensions": {
    "server_name": "000e00000b6578616d706c652e636f6d",
  },
  "certificateRequestExtensions": {
    "certificate_request_context": 0, 
    "signature_algorithms": "00020403"
  },
  "mutualAuth": true, 
  "extension-order": {
       "clientHelloExtensions": {
          Key_share
       },
       "ServerHelloExtensions": {
          Key_share
       },
  },
  
  "knownCertificates": {
    "61": "3082...",
    "62": "3082...", 
    "63": "...", 
    "64": "...", 
    ...
  }
}
~~~~~

ClientHello: 36 bytes = DH(32) + Overhead(4)

~~~
01                    // ClientHello
01                    // Profile ID
0020 a690...af948     // KeyShareEntry.key_exchange
~~~

ServerHello: 36 = DH(32) + Overhead(4)

~~~
02                 // ServerHello
26                 // Extensions.length
0020 9fbc...0f49   // KeyShareEntry.key_exchange
~~~

Server Flight: 80 = SIG(64) + MAC(8) + CERTID(1) + Overhead(7)

The EncryptedExtensions, and the CertificateRequest messages 
are omitted because they are empty. 

~~~
0b                 // Certificate
  03               //   CertificateList
    01             //     CertData.length
      61           //       CertData = 'a'

0f                 // CertificateVerify
  4064             //   Signature.length
       3045...10ce //   Signature
  
14                 // Finished
  bfc9d66715bb2b04 //   VerifyData
~~~

Client Flight: 80 bytes = SIG(64) + MAC(8) + CERTID(1) + Overhead(7)

~~~
0b                 // Certificate
  03               //   CertificateList
    01             //     CertData.length
      62           //       CertData = 'b'


0f                 // CertificateVerify
  4064             //   Signature.length
       3045...f60e //   Signature

14                 // Finished
  35e9c34eec2c5dc1 //   VerifyData
~~~


# Acknowledgments
{:numbered="false"}

We would like to thank Karthikeyan Bhargavan, Owen Friel, Sean Turner, Martin Thomson and Chris Wood.
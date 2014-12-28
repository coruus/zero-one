Multiple devices and key synchronization

So, Trevor asked Yan to get me to write up some tentative
thoughts on using multiple devices with modern messaging
systems. (These notes owe a great deal to the discussions
Yan and I have had, but any mistakes are entirely mine.)

Though I'm presently thinking mainly about mail-like services,
I've tried to make this note as general as possible, perhaps
at the cost of legibility.

# Introduction: The multi-device problem

So, we all have multiple devices. And any messaging design
should work as seamlessly as possible (given the particular
design's security goals) for this "multi-device" case.

# Preliminaries

(Skip ahead.)

Note: I don't cover the security of stored messages. This
is really important, but a topic for a separate mail.

## Definitions

A *user* is a human. An *account* is, well, an account
with a service provider. An *identity* is some mapping
between a set of accounts and a user or users. A *device* is
a platform-provided data compartment (this may be, e.g., a 
single physical device, a user account on a device, or a
browser profile).

A device is *enrolled* when it is able to send and
receive messages for an account.

## Framework

I'll assume the following, toy authentication and key
distribution/rotation framework:

*Providers* operate "account authorities", which sign
statements that attest to a device's control of an
account. (E.g., by ordinary user login authentication.)

There is a trustworthy *keyserver* which any account may
query to obtain the current set of keys associated
with another account. (More on this in another mail.)

And the following, with respect to keys:

  - Encryption and signing keys are separate; encryption
  keys are signed by a signing key.
  - Each account has, associated with it, a "recovery"
  signing key, whose private component is stored offline.

Keys may be rotated by cross-signing a new key, which
is uploaded to the keyserver.

Keys may always be revoked by a signature on a revocation
statement by the recovery key. (And in some of the options
I present, under other circumstances.)

## Security goals

Security goals are standard for a messaging system that
doesn't promise cryptographic deniability.[^traceability]

The adversary is the global active adversary; it knows all
internet packet's routes, can drop or replace them at will,
and will conduct any computationally feasible attack that
cannot be attributed to them. In addition, it has access
to the (possibly encrypted) contents of every server.

(I'm also very concerned about weaker adversaries: Script
kiddies, APTs, etc., but the GAA described here models
that threat reasonably well for this part of the problem.)

[^traceability]: In particular, signatures will be used
at least for some protocol messages. (In the particular
scheme I'm most interested in at the moment, practically
all messages will be signed.)

[^nearly]: Excluding some exceptional cases, to be
discussed in a future document.

## Local channels

A *local channel* is some method for transmitting secrets that
does not involve packets transiting the internet. (E.g., QR
code, Bluetooth, NFC, various more-or-less platform-specific
stuff.)

A *user-auditable local channel* is a local channel where it
is blindingly obvious to the user if there is a MitM attack
taking place. (E.g., it's obvious if someone sticks a piece
of paper with a QR code on it in front of your phone, which
is also displaying a QR code.)

A *one-way local channel*. Again, QR codes are the best example:
They only support data transmission in one direction. (So you
can't conveniently conduct an authenticated-key-exchange --
AKE -- with them.)

I'll assume that if a non-user-auditable local channel is used,
something like an SAS protocol (see infra) is used to conduct
a local AKE to prevent MitM attacks.

# Solutions to the multi-device problem

## Don't bother

This is TextSecure's current solution, AFAIK.

This is both an unacceptable user experience, and needlessly
compromises security: A user needs to bootstrap each trust
relationship (and suffers the possibility of being MitMed)
with every new device they use.

### Minimal option?

An idea for a minimal option for deniable messaging services:

Synchronize the known public keys of other users from device0
to device1 using a key derived from both a network channel and
a local channel.

Send the public key of device1 to device0; have device0 then
send that public key to all recipients that have a current
stored axolotl state.

(I've thought about this only a very little; so this may well
be a horrible idea.)


## Synchronize a single private keypair

A typical example is Whiteout.io: Encrypt the private keypair
under a strong symmetric key. In order to enroll a new device,
device0 displays a QR code containing the symmetric key for
device1 to scan. device1 then authenticates to the server and
downloads the encrypted private key blob.  See [Tankred Hase's
discussion on this list][wio_keysync] and Whiteout.io's [Github
repos][wio] for more details.

### The good parts

Really simple.

### The bad parts

**Prone to bad implementations.** The implementation
described by Whiteout.io on this list is characteristic:
 - Encrypted private keys are stored on non-US
 servers[^nonus]
 - The QR code *is* the symmetric key, so that anyone
 with <256 pixels on the screen[^qraont]

**Devices are indistinguishable.** It is impossible
to cryptographically distinguish the different devices
enrolled for an account.[^ind_and_tor] Several consequences
ensue:

1. Anyone who obtains the private key from any device can
impersonate the user. If messages aren't signed, they can
(usually) carry out KCI attacks against the account.
(So this is completely unsuitable for strongly deniable
messaging designs...)

2. It isn't possible to revoke (untrust) a single device.
In order to distrust one device, the user has to, e.g.,
sign a message with their long-term recovery key.

[^nonus]: So, e.g., the NSA has the encrypted key
blobs, and can distribute them to any member of its
"community" with essentially no legal limitations
under current caselaw.  And: If a user displays a
QR code in a public place, it is possible that, under
current law, they have no right to privacy in that
QR code. (So, think: airports, train stations, near
federal buildings, or anywhere vast numbers of 4k
video cameras might soon be deployed.)

[^ind_and_tor]: This could be useful if you are
building a system that uses Tor exclusively, but
pretty much useless otherwise. (Pond's "multi-device"
solution is essentially this: Synchronize the password-
protected state file to a new device. If devices
take locks on the file, this is safe.)

[^qraont]: QR codes should only be as legible as is
needed for them to work reliably. And 32-byte QR
codes are way too legible. This is easily fixed by
using an All-or-Nothing-Transform to expand the key
material. (512 bytes seems quite feasible for most
devices.) Question: Does anyone do this already?


## Separate signing keys, shared encryption key

Device0 is already enrolled. Device1 generates a new
signing key. Device0 and device1 authenticate a network
AKE using a local channel, and derive a shared secret
from both the local channel's shared secret and the
network AKE.

Device0 sends the private asymmetric encryption key to
device1 encrypted under the shared secret. The devices
(in no particular order) use the shared secret to authenticate
each other's keys, cross-sign them, and upload them to
the keyserver.

### The good parts

*All devices can decrypt all messages.* It isn't possible
for an adversary to, e.g., target malware to a specific
device by encrypting only to that device.

*Key compromise can be attributed to a device.* If all
messages are signed, a user can distinguish which
of a set of devices associated with an account has been
compromised. (They then may be able to use, e.g., a password
change and a message from a trusted device to revoke the
compromised device's signing key, without having to resort
to using -- and exposing to possible compromise -- their
long-term recovery key.)


### The bad parts

*The still-trusted devices may not have any key-material
to exchange a new symmetric encryption key.* Since they
have independent signing keys, they can, of course, do
key-exchanges to share the new private encryption key.
But this adds a lot of complexity to a little-used code-path.
(This occurs only when removing the corrupted device
makes the device enrollment graph disconnected.)

*All devices can decrypt all messages.* It's impossible for
an account to designate some devices as more trustworthy 
than others.


## Separate keypairs

Devices authenticate each other as above -- a local channel
plus a network channel -- but they just use the shared secret
to authenticate each other.[^messages_at_rest]

To send a mail to an account, the sender encrypts to all
encryption keys (with valid signature chains) returned
by the keyserver.

### The good parts

*Particularly important messages can be encrypted only to the
keys of particularly trustworthy devices.* Here's one possible
use-case: Suppose that I don't want someone with my phone to
be able to get both 2FA codes and password-reset emails for
the various accounts associated with my email address. I might
tell an encrypting proxy to not encrypt such mails to my phone.

*Recovering from a corrupted device is simple.* Since devices
don't share an encryption key, they don't all need to change
their encryption keys immediately.

### The bad parts

*Targeted malicious messages.* The flip side of all devices
being able to decrypt all messages, as above.

*Lots of keys.* N devices means N ECDH operations for the sender.
N devices means N times as large of a keyset to download from
the keyserver. Etc.

# Notes

## Fingerprints for manual verification of account identity

The most natural equivalent to a traditional OpenPGP
fingerprint in this setting would be some string derived
from the recovery signing key.

The best option is

    passkdf(salt=accountid, m=signingkey)

where `passkdf` is a sequential-hard KDF. Pace Joe,
preventing multitarget attacks is worthwhile.

## Short authentication strings

Trevor Perrin has described a very simple SAS protocol [on
this list][trevp_sas]. ZRTP uses an SAS protocol, but the full
description is split among many sections [of the spec][zrtp_sas].

A trivial modification to the protocol Trevor has described
can be used to increase the difficulty of mounting a successful
attack:

"Stretched" simple SAS:

    A = a*G
    B = b*G
      A->B: H(A)
      A<-B: B
      A->B: A
    K = H(a*b*G)
    SAS = passkdf(salt=A||B, m=K)

(This same modification is applicable to more intricate SAS
protocols.)

But I dislike SAS protocols when used over the internet: It is
rare that user agents take appropriate measures to detect (and
take appropriate countermeasures to) attacks.

[trevp_sas]: https://moderncrypto.org/mail-archive/messaging/2014/000036.html "Trevor Perrin. Short auth strings. Email to messaging@moderncrypto.org. 2014-01-31T09:24-08. (See infra for a slightly better approach.)"

[zrtp_sas]: http://tools.ietf.org/html/rfc6189#section-7 "RFC 6189 - ZRTP, section 7: Short Authentication String"

[wio]: https://github.com/whiteout-io

[wio_keysync]: https://moderncrypto.org/mail-archive/messaging/2014/000521.html "Tankred Hase. Whiteout secure PGP key sync. Email to messaging@moderncrypto.org, 2014-07-11T03:02-07"

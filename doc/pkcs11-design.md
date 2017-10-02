# Design for Remote PKCS #11

> *This is the (initial/proposed) design for the
> Remote PKCS #11 solution, to be built as an
> ARPA2 project.*

Some issues below haven't been fully worked out, and they are open for discussion and further thought.  They should be small enough to not interfere with an estimate of the work load, AFAIK.

**Prior knowledge:**

  * The [PKCS #11 standard](http://docs.oasis-open.org/pkcs11/pkcs11-base/v2.40/os/pkcs11-base-v2.40-os.html) for key storage.  Note specifically that everything allows the instant removal of tokens (such as smart cards) and slots (such as smart card readers) from the slot list covered by a PKCS #11 library; this is a nuisance to the application programmer, but it probably provides freedoms to the PKCS #11 Dæmon.

**Prior reading:**

  * "Security in a PKCS #11 Multi-Dæmon" and the
    [proof of concept](https://vanrein@github.com/arpa2/multi-pkcs11.git) for separation between
    PKCS #11 libraries.
  * "Merging PKCS #11 Object Namespaces" details how multiple PKCS #11 libraries can be layered to form one larger, virtual PKCS #11 service.
  * "ASN.1 Messaging Format for PKCS #11" describes the wire format (or at least its plaintext part)

**Contextual references:**

  * Be sure to understand the difference between *authentication* (who is it?) and *authorisation* (what permissions do you have?) which are conceptually different, and sometimes also implemented separately.  We allow an authenticated user to select an authorisation identity which may or may not be different.
  * [SteamWorks](http://steamworks.arpa2.net) is our configuration storage mechanism; we use [PulleyScript](https://github.com/arpa2/steamworks/blob/master/docs/pulleyscript/concepts.md) to map the data to tuple insertion and removals, which are passed on to a [PulleyBack API](https://github.com/arpa2/steamworks/blob/master/docs/pulleyback-api.md) for which relatively simple plugins can be designed for the various ARPA2 applications.
  * We use the ASN.1 language to specify network data structures.  ASN.1 has several binary and textual encodings of which we use the format DER (common in security applications), but there are also other mappings, such as for XML and JSON.  We have a [Quick DER](https://github.com/vanrein/quick-der) library for parsing and packaging DER materials rather comfortably; it comes with an `asn2quickder` compiler that produces C-style mappings for the structures (and we've come a long way with a Python embedding too).
  * The [PKCS #11 URI](https://tools.ietf.org/html/rfc7512) specifies data inside a given PKCS #11 repository.  Note that nothing is included about the library in use, or where it is running.  We will add that.  In general, users find public keys in LDAP, then reason that they want to sign with the accompanying private key, for which they find a `pkcs11:` URI that is unfolded into token selection criteria they use with PKCS #11.
  * Remote PKCS #11 is part of the [IdentityHub front-end](http://internetwide.org/blog/2016/12/21/idhub-3-services.html)
  * Note that IdentityHub allows a user to [inherit identities](http://internetwide.org/blog/2016/12/18/id-6-inheritance.html), which means that they can access private keys of groups/roles they belong to, and perhaps also for pseudonyms.  The IdentityHub supplies the Remote PKCS #11 dæmon with a list of alter egos, which are then resolved through Layered PKCS #11.  To make this work, we had to introduce PKCS #11 NAT as well.
  * As party of the inheritance infrastructure of the IdentityHub, it is possible for users to "drop privileges" by switching pseudonym or adopting the restrictions of an alias.  To get there, authorisation is needed, which involves an inquiry over Diameter/SCTP at login time.
  * The IdentityHub is a multi-tenant service, targeted at domain hosting.  It contains a key management process, which creates and destroys keys and certificates for domain users.  It will be authorised to do this on Remote PKCS #11 on behalf of users.
  * Mozilla is very explicit about its support of PKCS #11.  You can add "security devices" which are PKCS #11 libraries, and the user can select certificates off of it.
  * Kerberos is a system of shared secrets that are dispensed by a realm's KDC.  We rely on Kerberos, and even have plans of using [Kerberos over NFC](https://github.com/arpa2/kerberos2nfc), to easily pass certificates to mobile devices.  The common interface over which Kerberos is used is [GSS-API](https://tools.ietf.org/html/rfc2743) which we will use too; an alternative might be our own work on [TLS-KDH](http://tls-kdh.arpa2.net) which offers a rather modest TLS implementation.


## Naming the Game

Remote PKCS #11 is just a technical name, which nobody will like.  We need
a better one.

This project will be called **keehive**.  Nobody knows this word yet, not even
big G or big M.  And everybody who pronounces it can tell what it does, sort of.

So, the PKCS #11 library on the client would be `libkeehive.so` and the dæmon
on the server may be called just `keehive` and we can use many terrible puns;
as long as they don't interfere with pragmatic usability of the tools and code.

Obviously though, the main process for the keehive could be called `queen`, and
any processes in its PKCS #11 backend libraries would be her `drones` or
`workers`.


## User Experience Design

PKCS #11 applications tend to allow the configuration of a library, selection of a token inside it, and configuration of a PIN for access.  Take a look at Mozilla's "security devices" for a graphical example.

We want the user to be unaware of the PKCS #11 layering, so the IdentityHub can add group memberships and by that, any keys available to the group, without the user's prior knowledge.  Also, removal of a PKCS #11 layer for a group should not involve any user action.

This is why we want layering for PKCS #11, even though the PKCS #11 model allows for access to multiple libraries and multiple tokens under management of each library.

We do want users to select their authorisation identity, or at least should they be allowed to do this.  This could be useful to "step down" to an alias or to migrate to one of their pseudonyms, so as to constrain the possibilities over the given link.  There are a few places where this could be done: (1) based on the slot description, which should maybe mention Remote PKCS #11 instead, (2) based on the token label, which is a perfect place for a user identity, (3) in the PIN (see below).  The token label would seem to be the best place, as that is usually selected when a Remote PKCS #11 library is setup in an application.  **TODO:** It also occurs in the `pkcs11:` URI scheme, which may or may not turn out to be helpful.

We want the user to have software installed for access over Remote PKCS #11, and access the public keys as part of their Single Sign-On through Kerberos.  The service used will be `pkcs11/host.name@REALM`, which is to be configured as part of the Remote PKCS #11 setup.

For mobile device users, we have a future project in mind that stores a fixed secret on the device, but without any power other than decoding Kerberos tickets passed to it.  We imagine NFC as a transport for passing this, so the device can be tapped against a token-granting pad on the wall, or possibly a small keychain device is involved as a sort of key.  This way, an easy action loads the SSO base credential onto the mobile device, and only the receiving device can decode it; after a limited time, the SSO credential expires, to reduce the impact of device loss.

## Coding

This is a project for C, as a result of that being the primary language for PKCS #11, and we should try to avoid too many intermediate layers.  Garbage collection is also a problem; PINs should be zeroed after use and before deallocation, whether they are stack-based or heap-based.  As an alternative to C, we might choose C++ but that is requires a stringent, security-supportive subset.

We develop for Debian Stable, which is a slowly moving target.  We want IdentityHub components delivered as Docker components, though we may find someone to coordinate that for the various subprojects.

Our building environment is CMake, for which Adriaan is a very good source, though I'm also fairly good at it, and the others also use it.

We communicate over IRC, through `irc.arpa2.org` port 6667, channel `#identityhub`, though we might add a `#pkcs11` if you like.  We do not aim for interactivity, though when called out by our username we tend to respond faster if practical.

We publish our software on GitHub, in the ARPA2 project.  Let Rick know if you need projects added or otherwise managed.

## Components to Build

There are a few parts that make Remote PKCS #11 tick:

  * On the **client**, a Remote PKCS #11 Library that encodes API calls into RPC
  * On the **server**, a Remote PKCS #11 Dæmon that takes in RPC calls; participates in authentication and authorisation; is aware of the layering for each connected authorisation identity; passes RPC calls on to their respective backends
  * In the **backends**, a wrapper around a PKCS #11 library; there may be a need to reload these dynamically when new tokens have been added
  * For the **IdentityHub** integration, pulling information and delivering it to local configuration mappings.
  * Integration with **SoftHSM2**, possibly by extending its code to form a dedicated and trusted backend for the Remote PKCS #11 Dæmon.  The integration involves creation and deletion of tokens in response to added and deleted user/group/role identities in the IdentityHub.

We already prepared a number of tools for the integration with the IdentityHub.

## Protocol Design

The protocol for Remote PKCS #11 is an RPC variant on the PKCS #11 Base Specification.  Every call is split into a request and response, with an ASN.1 format for each.  It is deliberately designed to be suitable for publication as an RFC.

PKCS #11 is designed as an API, meant for local use.  Using it remotely is only safe when it is run over a security layer.  TLS is a less likely candidate, because Remote PKCS #11 is meant to service TLS, among others.  Passwords are also less suitable because this would store long-term credentials on clients, which probably will include ill-protected mobile clients.  What we need is a key derivation system that is there to last.  This already exists, and in a way that suits the central management in an IdentityHub: Kerberos, which is most easily used over GSS-API.  An alternative may be TLS-KDH.

The following protocol layers are good to support in a switchable manner:

  * **RPC** is a mapping of PKCS #11 API calls to/from DER encoding; the encoding should be switchable so future extensions are possible.  It may prove efficient to parse the RPC format only partially in the Remote PKCS #11 dæmon, to accommodate validation and routing only, and have the rest of the work done in the layers formed by PKCS #11 libraries.  (An alternative to partial parsing might be full parsing, and passing along the parsed [overlay structure](https://github.com/vanrein/quick-der/blob/master/USING.MD) with the data being pointed into.)
  * **Secure Layer** encodes the RPC format between client and server in a secure manner.  Secure means authenticated and encrypted for the Remote PKCS #11 Service.  The initial design would cover GSS-API, but future switchable alternatives may be added for TLS-KDH.  With Kerberos, we can and should use mutual authentication.  (**TODO:** We might use a slightly different format for asynchronous messaging such as over AMQP, where we provide the might authenticate while providing a [subkey](https://tools.ietf.org/html/rfc4120#section-5.5.1) and follow with [information encrypted](https://tools.ietf.org/html/rfc4120#section-5.7) under that subkey.  Not sure if this also works in GSS-API, but it probably is.  Only the proper recipient could read/process this kind of submissions, but there would be no need for a session concept.  Also have a look into [response formalisms](https://tools.ietf.org/html/rfc4120#section-5.5.2) though; this should probably use the KDC-supplied session key but mention the same subkey, to show that it mentioned to decode it.  The sender can store a hashed version of the subkey, which is more secure than storing it literally.  The reply can once more encrypt the added part with the subkey.  We might also decide to use this mechanism for all traffic.)
  * **Transport** is the client's choice of TCP and SCTP.  In addition, we shall use AMQP 1.0 because that allows for queueing of work while a component is down, which is incredibly useful in terms of operations of the complex IdentityHub.  We should start with TCP, the rest is extra and may be captured in a generic dæmon rather than specific for the Remote PKCS #11 Service.  TCP has no framing, so we should either recognise the DER length after [loading the initial 5-6 bytes](https://github.com/vanrein/lillydap/blob/26c1bbea1cf9d40f33ea5bba7450ec52d12367dd/lib/derbuf.c#L41), or we could have less self-respect and load a big-endian 32-bit number with the length that follows.  IMHO, the second really needs to be argued for.  With SCTP and AMQP we have framing, so we need not worry about this.  SCTP allows requests to arrive out-of-order when this is more efficient, so it improves concurrency.  Note that SCTP can be run over [UDP tunnels](https://tools.ietf.org/html/rfc6951), using port 9899 on both ends; this is integrated in the socket API, but individual sockets either use a tunnel or not.

*Suggestion:* It may be easier and more interesting work, and certainly more awesome, to make an RPC mapper from ASN.1 to stub/skeleton code in C, using a Python script.  This would match the style (but not necesserily the evolved structure) of `asn2quickder`, and could be added to Quick DER as a generic tool.  Such an RPC mapping would need to take the mentioned partial mapping into account, perhaps based on the names and types in function prototypes.

## Security Design

The structures to make the Remote PKCS #11 dæmon secure.

### Separating Processes

The PKCS #11 libraries are usually coded by independent parties, which happen to be in competition.  It is not desirable to share process state between multiple such libraries, so each should be encapsulated in separate processes, and be sent only their set of proceses.

The reason for the separation of libraries into processes is to avoid security concerns (or worse, not taking them into account) with hosting providers when they are faced with customers who want to add their own PKCS #11 library for their domain.  Separating the libraries in different processes means that the libraries cannot take rogue peeks into each others' interiors.  It effectively means that users get more choices when configuring their domain security.  This is the reason for the proof-of-concept in the `mutextest.c` test program.

There is no need to split the uses of a PKCS #11 library (version) to access a variety of tokens; a given PKCS #11 library (version) may be assumed to have its own interest at heart, and keep various sessions for various users open at the same time.  Libraries can open multiple sessions, and should have no problem opening them on various slots.  Slots can each hold zero or one token, and a token represents the key material of an individual user or group.

**TODO:** The interaction between sessions is confusing, since rights are shared.  It looks like it will be necessary to have a separate process for readonly use (`CKS_RO_USER_FUNCTIONS`) and read/write use (`CKS_RW_USER_FUNCTIONS`) but the latter would only work when permitted by Diameter/SCTP; for instance within groups, a member may not always be granted write access; write access means that keys may be added or removed.

Layering combines multiple PKCS #11 libraries, and this can only be done in the overall Remote PKCS #11 dæmon.  Since each library can pick its own `CK_OBJECT_HANDLE` values, it will be necesssary to create a mapping.  There is no reason why a library-serving process cannot make its identities map into a reserved range, but that would require the overal dæmon to check whether they do not go outside their allocated ranges.  It may be easier to do it all in the central component.

## Read/Write Access to PKCS #11

The functions that are permitted to a session are determined by their type, which is set when a session is opened, and which help to determine details of the `C_Login()` operation.  Please take note of the somewhat confusing interaction that sessions may have.

A client authorises with a certain identity; this identity defines the PKCS #11 backend library that is accessible in a `CKS_RW_USER_FUNCTIONS` session.  Any additional identities, including group identities, are accessed under `CKS_RO_USER_FUNCTIONS`.

As an exception to the above, some group members may be granted write access, when they authorise as the alias that is registered as a group member.  Note that a group member has their own identity, so `john+chef` may be the alias that is a member of `cooks` with membership identity `cooks+johann`, so unrelated beyond a mapping from membership identity to user alias.  Those membership identities that are granted write access, may be able to create key material for the group.  Similarly for roles and role occupants.  **TODO:** Would this be useful, would it be used?

**TODO:** There does not seem to be a need for token administration, so there is no need for `CKS_RW_SO_FUNCTIONS` sessions.

## Authorisation

The main step in authorisation is the secure transport layer, which delivers the following kind of credentials:

  * `pkcs11/host.name@SERVICE.REALM` for the service
  * `user+optext@CLIENT.REALM` for the client

These will be mutually authenticated.  Key material will have been exchanged.  The `SERVICE.REALM` and `CLIENT.REALM` are the uppercase form of a domain name, and [there may be reasons](http://realm-xover.arpa2.net/kerberos.html) for the two to differ.  Kerberos would speak of client and server principal names, where we tend to speak of client and server identity; we would map the realm name to lowercase for that, and use a special notation for the service (which is not needed here AFAIK).

Every token resides under a single realm, namely that of the client.  It is identified by the client identity, so a mapping from client identity to a backend PKCS #11 library is needed.  The mapping is not used with the authenticated however, but authorised identity.  It is also used for any additional identities.

The `C_Login()` call is surprisingly simple; it constitutes of entering a PIN, but not a user name!  The idea is that a token is bound to a user, after all.  But what we would like to do, is allow the user to select a user identity for a particular use, and be authorised for that.  It is not practical to demand this from the user's Kerberos setup.

Since we run over a secure transport, and authenticate the client over that, we can safely assume that the client is who we want it to be, founded on much better security than the PKCS #11 PIN.  As a result, the PIN is a "free slot" for us to use for exchanging information.  Tools that use PKCS #11 commonly request a fixed, textual secret and pass it on literally.  The standard imposes no restrictions, so we will use a format such as `authz_as_user+fwd_pin_data` where the `authz_as_user` is the user identity to authorise as.  The Remote PKCS #11 dæmon makes a backcall over Diameter/SCTP to authorise the requested change of identity from the one provided over Kerberos to this one.  **TODO:** There is no reason to support for a change of domain/realm.  **TODO:** It may be a better idea to put `authz_as_user@domain.name` in the token label, and reserve the PIN field for `fwd_pin_data` (and allow the `+` symbol to occur in it).

The central control over PKCS #11 from the key management service in IdentityHub also follows the authorisation mechanism; its master access would however be configured instead of require an authorisiation backcall over Diameter/SCTP.  Sort of like a `root` user for a given domain, but as a Kerberos principal name for each realm.  **TODO:** These sessions are also mere `CKS_RW_USER_FUNCTIONS` sessions; any token administration will be handled by independent software operating on the PKCS #11 backend libraries.

## Access to PKCS #11 Backends

Even though we shrug on the PIN mechanism, the loaded PKCS #11 libraries still insist on getting one.  We make the explicit assumption that the Remote PKCS #11 Dæmon controls the token's use entirely (except for token administration, e.g. PIN replacements) and that nobody else is adding or removing objects.  We also assume that the Remote PKCS #11 Dæmon can choose the PIN to be set on the token.

As a result, we shall need to construct a PIN from the user's access attempt.  We may use a part of the PIN for that purpose, namely the `fwd_pin_data` shown above.  We should device a clever algorithm that does the following:

  * Represent `fwd_pin_data` in a textual form without embedded `+` characters
  * Combine with information stored in the Remote PKCS #11 Dæmon to find the backend PIN
  * Result in different `fwd_pin_data` for each client
  * Incorporate information to describe the client, to prevent easy reuse in other places
  * We need to reduce to a PIN that conforms to the backend's ideas of `ulMinPinLen` and `ulMaxPinLen`.
  * Prefer not to use UTF-8 in PINs in and out; it wastes bandwidth and introduces alternate forms for the same value; we shall limit ourselves to 7-bit ASCII, and perhaps to base64 with a substitute for the `+` character; we would hit 6 bits of entropy per character; so when `ulMaxPinSize` is as low as 8, we can have 48 bits (but usually it's much better).
  * **TODO:** Support explicit retraction of any given `fwd_pin_data`?
  * Perhaps: Validate the resulting PIN before sending it to the PKCS #11 backend library?

**TODO:** Define an algorithm.  This should be doable with some clever tricks with secure hashes, challenge/response and proper field formatting.  This is a cryptographic design, so Rick's chore.

We now have a setup where the same PIN is generated for the same backend, but with variations in the sources that supply it.  Note that there may be additional data stored for authorised layers (like for group access provided to group members).  This may well be stored in the same tables that store the relation between a user and their accessible tokens.

**TODO:** We shall need some place where the token PIN can be found, or we would need a token access to grant a new user to access it.  Otherwise, the proposed cryptographic scheme is too restrictive.  As a strong point, this store of token PINs would be out of reach of the public-facing Remote PKCS #11 Dæmon.

## Relation to IdentityHub

This software might be useful on its own, but we do need some specific hooks to integrate with the IdentityHub.  The following aspects are integration concerns:

  * Users, groups and roles that need to be supported with a PKCS #11 token; addition and removal of these leads to addition and removal of their PKCS #11 token; in the case of SoftHSM2, there is a need to create and destroy tokens; for others, the token information must only be installed into the Remote PKCS #11 Dæmon databases
  * Authorisations of users to go from one identity to another (dropping privileges to an alias, or changing pseudonym); there is an explicit mapping from a domain name to its manager, charged with key management
  * Mappings from authorised user identies to the identities that may be added as extra layers
  * Mappings from user identities (all the layers) to the PKCS #11 backend libraries; their library, `CK_TOKEN_INFO` and possibly even their `CK_SLOT_INFO` structures.  For remote PKCS #11 applications, a description of the location and protocol information (supporting at least the current Remote PKCS #11 protocol, and taking care to support at least the described variations).
  * For any given token, there will be a need to store handling information for PIN-based access to the PKCS #11 backend libraries.

We should design this together, I suppose.

As a hint, the mappings are key-value lookups.  We have used BerkeleyDB for these uses.  Even if it is not the fastest option, it really is rock-solid, and comes with advanced features such as replication.  We have
[created BDB mappings](https://github.com/arpa2/tlspool/tree/master/pulleyback) to implement the [PulleyBack API](https://github.com/arpa2/steamworks/blob/master/docs/pulleyback-api.md) which receives configuration changes from the [IdentityHub structures](https://github.com/arpa2/idhub-service) that we tend to create with our user-facing frontends for managing their domain.

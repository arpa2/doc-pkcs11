Layered PKCS \#11 — Architecture
================================

>   *Layering PKCS \#11 is a nice architectural ideal. How can we establish it,
>   knowing how stubborn PKCS \#11 can be?*

We are assuming that Remote PKCS \#11 can be resolved in a reasonably boring
manner.

-   We can map all calls and their returns to ASN.1 and pass that over a
    suitable transport. Packing and unpacking can be subject to boringly concise
    rules, and should work with a high degree of reliability, given the strong
    alignment in the mind sets of ASN.1 and PKCS \#11.

An easy way out is to decide that all will be done based on one software PKCS
\#11 implementation

-   Strong integration with SoftHSMv2 probably helps us gain efficiency, improve
    hosting, and bypass the funnelling needs of the PKCS \#11 API on the hosting
    platform.

-   Ideally however, we would layer PKCS \#11 APIs on top of each other.

Make ACL decisions on the grounds of a user’s transport identity. They might use
TLS-KDH, or ECSRP, or other mechanisms.

-   The PIN is helpful for encryption. Do we pass it on to sub-layers? It seems
    better to use the PIN to encrypt lower-layer PINs instead, so each PIN can
    be different. Ideally, we would avoid that the hosted stacking layer gets to
    see the backend PINs, as it could be a MITM.

-   Some PKCS \#11 layers, notably the locally hosted ones, could use the PIN
    directly, by having it delegated to them.

The structure is based on a few databases.

-   Given a logon identity, lookup available PKCS \#11 tokens and layers, and
    present them during `C_GetTokenInfo()` as one of the options. Each is a
    composition of PKCS \#11 layers, and it is the task of the Remote PKCS \#11
    server to serve out these choices.

-   The served tokens can be opened at will, as is common with PKCS \#11. Their
    identity is internally described with a 128-bit random number such as a
    UUID. Such UUIDs helpt to quickly establish a backend tokend shared among
    multiple users and/or sessions.

-   The layers of a served token must be composed; high bits may select the
    internaly layer and the remainder would be the backend-specific object
    identities; a mapping may be needed to override a sane default.

-   Can users plugin their local PKCS \#11 implementations, e.g. en USB token,
    as one of the layers? Principally yes, but there would need to be a backcall
    mechanism. Note however that `FunctionCall` and `FunctionReturn` are
    separate types, leading to no confusion when backcalls are sent down the
    normal upcall channel.

-   Each token’s UUID points to its backend implementation, which could be a
    SoftHSMv2, but in that case with a database to hold the objects.  Those
    databases may be replicated for stability and reliability.  The derivation
    of `CK_TOKENINFO` and from the UUID is more useful than when we had done it
    in the other direction.

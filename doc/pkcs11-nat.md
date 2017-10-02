Merging PKCS \#11 Object Namespaces
===================================

>   *The term Network Address Translation has become synonymous with dramatic
>   attempts to keep IPv4 afloat past its expiration date.  The mechanism of
>   folding multiple namespaces into one is one that has more fields of
>   application however, and PKCS \#11 could also benefit if it were applied to
>   object handles.  Luckily, without the downsides of NAT in a transport layer
>   that considers mappings transient, and unrelated to sessions.*

When dealing with shared identities, it can be quite useful to share a PKCS \#11
object namespace; and when it is additionally possible to add personal
modifications, then it is quickly necessary to either exercise authorisation on
individual objects, or to combine PKCS \#11 object namespaces.  As it turns out,
this is quite possible with due to how the PKCS \#11 specification handles
objects.

Properties of Object Handles
----------------------------

In PKCS \#11, objects are references as object handles.  These happen to have
great potential of being used in a “NAT” style, that combines objects from
various namespaces:

-   Object Handles are opaque identifies that happen to be represented as an
    integer type.  The handles are always initiated by the PKCS \#11 library and
    applications must handle them without modifications.  This means that a
    mapping between a “NAT mapping” of internal object handles to external ones
    can be easily maintained.

-   Object Handles are local to a session.  Meaning, their duration is precisely
    bounded and so the moment of cleanup for “NAT mappings” is a reliable one.
    Also note that the locality of object handles to sessions implies that the
    mapping can be kept local to a session, instead of being shared and fixed.

-   The occurrences of an object handle are well-defined throughout the PKCS
    \#11 library and the definition is complete; there is no way to sneak an
    object handle into an opaque structure and bypass the “NAT” layer.  No need
    for “helper apps for FTP” in this well-defined case!

-   The size of an object handle is defined to be at least 32 bits.  This is a
    considerable space for the application domain, and can be thought of as
    “should be enough for everyone”.

Applications of Merged Object Namespaces
----------------------------------------

PKCS \#11 is normally used for a single user at a time.  When groups and roles
enter the playing field however, we need a scheme to share the objects held.
The only way to do this in current PKCS \#11 is to share the access to the same
object namespace.  We may hope that our vendor has implemented a form where at
least each user gets their own PIN — but we usually hope in vain.

This can be a real problem when personal additions or variations to the contents
of the object namespace are useful.  For example, in a setup where a user may
use personal certificates, as well as those from a group or role he is engaged
in.  We end up cloning much of the content across PKCS \#11 stores, which defies
the security model (making the lowest threshold the definition of the overall
security level), and it has its own ramifications in terms of management of key
rollovers.

Contrast that with a situation where a group may have one PKCS \#11 object
namespace, and an individual can have its own.  The two object namespaces are
merged to appear like one.  When at least the group namespace is managed
remotely, we can leave its security as-is.

This also brings the potential of trimming down the access that various
applications have to a PKCS \#11 store.  Some may share keys and others may not.
It may be possible to add layers as readonly, or as read-write.  Layers could be
setup to blind the objects in other layers.

Implementation of Object Handle Translation
-------------------------------------------

The term “NAT” does not apply in our context; we should speak of *Object Handle
Translation* instead.  As will be clear, the scope of translation is a session.

To be as widely usable as possible, we shall use 32-bit identifiers for our
HSMs.  These identifiers will be split in ranges, defined by a prefix of those
32 bits, that help to make the mapping on ranges of backend objects.  A nice
prefix size might be 16, for example.  When we have only one backend, we can set
a prefix size 0, which means that backend identities can be translated directly
— if they fit in 32 bits too, that is.  Otherwise we will use the mapping to
trim the range.

Note how this takes advantage of the common practice of PKCS \#11 libraries to
have some form of sequential numbering for object handles —be they determined by
the physical store or by session state— in needing only one prefix for most
common backends.  In fact, backend that stubbernly deviate from this assumption
may receive a more dedicated translation that actually assignes object handles
sequentially within the current session.

-   The general mapping now is from a “published” 32-bit object handle to an
    N-bit internal handle (N being at least 32, as per the PKCS \#11
    specification).

-   An exceptional mapping is made for the invalid file handle value 0, which is
    of course between all backends.

-   A postfix of M bits (M\<=32) will be the same as a backend’s M bit postfix.

-   A prefix of N-M bits in the frontend will map to a backend and the backends
    object handle size minus the postfix size M.

-   Backend object handles’ prefixes (of their size minus M) are used to lookup
    their allocated N-M prefix in the frontend; if none exists yet, then a new
    entry is created in the frontend prefix allocation.  Failure to do so is
    reported as a memory/size/allocation problem.

Running C\_FindObjects
----------------------

When a new C\_FindObjectsInit is called, it creates a local management
structure, which includes an iterator for the backend being interrogated.  When
one backend ends the search, the next will be tried.  There is a possibility of
searching backends in parallel, if so desired.

While iterating, object handles are passed back to the frontend, which
translates them (possibly allocating new prefixes).

There is a possibility of “blinding” certain backends.  This means that a
successful C\_FindObjects in one namespace would avoid results in another name
space.  This can be helpful to avoid confusion between the uses of layers.  (But
it also might be unsettling when other search criteria suddenly finds objects —
or is that to be expected when search criteria vary?  A matter of taste, I
suppose.  One way of making this more sollid is to fixate blinding rules
explicitly configured instead of caused by the search criteria used by an
application, but that is also much more complex.  Perhaps this is a growth path
for this solution, and it might be ignored initially.)

Blinding is certainly useful when we permit C\_DestroyObject on layers that we
have no write permission to.  We might however use CKR\_ACTION\_PROHIBITED as
well, and manipulate CKA\_DESTROYABLE when passing through (and, of course,
CKA\_MODIFIABLE for C\_GetObjectAttributes gets the same treatment).  This also
indicates that growth is possible, but not immediately necessary, where blinding
is concerned.

Creating keys and key pairs
---------------------------

When invoking either C\_CreateKey or C\_CreateKeyPair, the backend will report
object handles for the newly created objects.  These will be passed through the
mapping as might be expected.

A question that will rise is where to plant the new objects, or in other words
what backend should be selected to conduct the work.  It is probable that a
primary object name space can always be appointed, or perhaps it is the only one
that supports searches.  In this light, note that the mechanism support shared
object name spaces, and choosing a suitable one just for creation of a key may
be a sensible angle to take.

Additionally, any explicitly configured blinding rules might have something to
say about where to create new objects.

Finally, PKCS \#11 has wrap/unwrap mechanisms, which might be used to move
objects between key stores, if it turned out that keys were created in the wrong
places.

Offering Tokens
---------------

PKCS \#11 identifies tokens within a library.  Identifying differences between
the tokens in a library are its label and serialNumber, and perhaps even its
model.  It should be quite useful to the application using the frontend when the
label gives a clue about the merged object namespaces offered.

The serialNumber could be a key into the database, except that it is not
repeated in the call to C\_OpenSession.  In fact, since the SLOT\_ID is
presented and this number may be more helpful to find the description of the
token.

The various combinations of object namespaces that could be offered to the same
user should all have different labels and/or different serialNumbers, so as to
make them distinctly selectable.  So the serialNumber is mainly a place to store
a unique value that may or may not be derived from identifying elements outside
what is shown in the CK\_TOKEN\_INFO and/or CK\_SLOT\_INFO.

### Protecting label and serialNumber

If the label (and perhaps the serialNumber) are subject to privacy concerns, or
if a complete listing of all tokens that may be subject to login could go
rampant, then it can be helpful if the transport is already somewhat selective.
Since the transport usually involves a security layer, this is often possible:

-   When using local machine knowledge, the PKCS \#11 library is called form a
    known user, and this can be used to select the visible names; this is only
    secure when the interaction is local to the machine, or perhaps when it is
    trusted by a remote machine and manages to indicate the user in a secure
    manner as part of the remote PKCS \#11 transport.

-   When using TLS for Remote PKCS \#11, then the ServerNameIndication can help
    to offer a limited range of slots/tokens; not having any security attached,
    this is merely of interest to shortening the list for reasons of efficiency.

-   When using TLS with client authentication or a Kerberos-based transport for
    Remote PKCS \#11, then the client identity can be used to select available
    slots/tokens.  This does have a security implication.  When searching a
    database for slots/tokens, it is possible to search for the combination of
    the authenticated client identity, and assign slot numbers based on their
    order of appearance in the database.  Since this order may change over time,
    the data from the database should be copied into the frontend before
    responding with CKR\_OK from C\_GetSlotList; in other cases, PKCS \#11
    provides no guarantees to the caller, because tokens and even slots may be
    inserted and retracted with much the same spontaneity.

### The role of Authorisation for the Token Lifecycle

Authorisation can be used to construct a list of slots/tokens to be used.  This
is especially true when the application’s user can be authenticated.  Based on
an authenticated identity, it is possible to find resources to which a user is
authorised; for instance, Remote PKCS \#11 tokens assigned to the user as well
as his aliases, pseudonyms, roles and groups.

It may happen that authorisation indicates authorisations to tokens that have
not actually been created.  It may be possible to create such tokens on the fly,
especially when all the information needed for this repository can be derived
from the context.  Such lazy creation of tokens can help to simplify the
coordination between various bits and pieces of an infrastructure.

Like lazy creation, it may also be beneficial to have automated retraction of
tokens.  This may simply derive from changes in authorisation rules, or it may
be part of regular housekeeping jobs.  (One might pose security concerns when an
identity is removed and then immediately recreated, keeping the PKCS \#11
repository with all its contents might not be the intended purpose.)

Database Model
--------------

-   We assume an authorisation model that maps a client identity to a list of
    identities as which they may act.  Some identities may be readonly, meaning
    that no new identities can be created (such as introducing new group
    members).

-   We assume a static mechanism to create a token label and serialNumber for
    each listed identity accessible to a client identity.

-   We assume a mapping from each listed identity to a set of object namespaces
    to be merged; each object namespace is assumed to be a PKCS \#11 repository
    that we have access to
    (TODO:HOWTOKNOW?:HOWTOENFORCE?:LOCALONLYFORNOW:FUTUREKERBEROSMAGIC)

-   We assume a mapping from each object namespace to an underlying store,
    perhaps through a PKCS \#11 implementation such as SoftHSMv2.  We assume
    that any issues such as backup and replication are handled at that level.

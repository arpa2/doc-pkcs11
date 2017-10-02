Security in a PKCS \#11 Multi-Dæmon
===================================

>   *Having one daemon handle various PKCS \#11 libraries naively could be an
>   open invitation for rogue libraries to abuse other libraries. We need to
>   keep the libraries separate through a kind of backend dæmon. What mechanisms
>   would be suitable?*

Here, we turn to mechanisms that can help to have an efficient infrastructure
with “pre-loaded” PKCS \#11 libraries connected to a dæmon that serves many.
Important in doing this well is to keep it protected. Separation of the main
dæmon from library backend processes appears to be called for, and some form of
IPC, encryption or Linux’ cgroups might be used to communicate between the main
dæmon and the library backends.

Requirements to the multiP11 design
-----------------------------------

Let’s define a working name for this collection of processes; we shall call it
the multiP11 dæmon. It’s design may be complex for reasons of on-system
security, but it should run on POSIX or, if that is impossible or unpractical,
then at least on standard Linux.

### Reliance on root privileges

It is acceptable to assume `root` privileges for its setup, so long as the dæmon
drops those privileges before being available to the outside world. There may be
a master dæmon to process internally originated configuration changes, and that
may fork off other worker processes for any new connections; once again, these
workers should drop `root` privileges before they connect to the network.

### Reasons for keeping libraries separate

When used in full, the multiP11 dæmon may host for multiple domains, each under
different control. Each of those domains should be able to bring in libraries
without affecting the security of others. In general however, a
binary-distributed PKCS \#11 library may hold unsuspected rogue code, and should
be kept from accessing other libraries’ name spaces.

This is especially true because the multiP11 dæmon is likely to keep sessions
open to each of the libraries, for reasons of efficiency and perhaps to support
client applications that assume long-lasting sessions. A final reason may be
that we could consider future protocol versions that demand such long-lasting
sessions; think of transaction support (meaning, rollback facilitation) and
event notification (meaning one client application makes a change and other
client processes are notified).

### Differentiating PIN codes

One PKCS \#11 library should not be able to access other PKCS \#11 libraries,
and that means that each library should have a separate PIN. This is difficult
because the client application will only offer (at most) one PIN.

### Future-proofing for Change Notifications

Future versions of Remote PKCS \#11 may include support for change
notifications. This is a missing facility in PKCS \#11 libraries, but it can be
really useful, especially when multiple client applications can access the same
content from various locations. The multiP11 dæmon might assume that it sees all
traffic accessing a service, and send any new instances created by one client
application to others that subscribed to it. Subscriptions would be quite
similar to object searches, with the modification that they would wait, possibly
indefinately, for matching results. So, `C_FinObjects()` becomes
`C_AwaitObjects()`. Note that this does not support notifications of object
removal yet.

### Future-proofing for Transactions

Future versions of Remote PKCS \#11 may include support for transactions. To
facilitate this, we need to be able to rollback operations. This can be
facilitated with the existing PKCS \#11 library functionality, so it is even
possible to do this on multiple PKCS \#11 libraries at the same time.

Transactions would need to be explicitly started to ensure the recording of
rollback information. When adding objects, they would not be shown to other
client applications yet, and when removing objects, they would already be taken
out of sight of the requesting client application.

This requires that a filter is built when editing objects; one filter that
indicates that certain objects are visible to *only* one client, and another
filter that indicates that certain objects are visible to *all but* one client.
When a transaction materialises, these filters are taking out, and/or their
respective objects are removed.

The multiP11 dæmon should be able to establish the rights to these changes by
looking at session properties and object attributes. Note that it is very unlike
PKCS \#11 to permit locks on objects; tokens may be torn out of their slots at
any time, is the general reasoning. Transactions may fail however.

Note that collected errors during a transaction can permit us to continue to
send actions that are simply refused until the attempt to commit; of course
transaction-collected errors may also be observed and cleared explicitly.
Clearing might be supportive of continued invocations if it can say things like
“if we just went from `CKR_OK` to `CKR_xxx`, then ignore the last operation’s
error response (and skip the next *n* calls?)”.

Although the collection of errors in transaction state may sound like support
for bad programming, it actually implements tedious code that all proper
implementations need to write. And what’s more important: it enables the
construction of batch operations which are shipped to Remote PKCS \#11 as one
large transaction. Doing that completely however would require a way to insert
variables into the scripts… by saying “I will refer to the result of the next
operation as `OBJECT_HANDLE 30` in what follows”, for instance. Could that be
yet another use of the “NAT” layer in multiP11?

Structuring PIN Separation
--------------------------

We need to derive different PINs for each library loaded by the multiP11 dæmon,
but source it from one and the same source PIN, presented by the user. In doing
this, we should be supportive of a certain security level; the examples below
assume a 256-bit security level.

### Not caring about PINs from Client Applications

One reasoning might be that we don’t care about PIN codes at all. We are
assuming a secure transport, presumably built on top of Kerberos-authenticated
access, as that helps us to separate out clients, get to their authorisation
profiles, and help us decide what backend libraries to offer.

With this in mind, we might consider the PIN supplied by the client as a
formality; in fact, it may be used to reveal information, such as the choice of
a service description (perhaps the name of an alias, group or role) within the
authorised sphere of services (or identities).

### Deriving PINs for individual PKCS \#11 Libraries

Had we been dependent on a PIN supplied by a client application, then we would
have proposed a HMAC that used the PIN as a key and applied it to a
library-specfic challenge; we would then XOR the result with a corrective
pattern specific to the library, and possible interpret the outcome as a
character string that might be cut down to a smaller size on the occurrence of a
specific invalid UTF-8 sequence at the end. But this is not necessary, as we
need not care about the PIN from the client application.

What we might do, is use the fact that we are running over Kerberos. This means
that there is a server credential, usually in a keytab, that we may use to
decipher things. This can be used to decipher the credentials of the various
backend libraries. The libraries may insert it as a challenge in a public
session data object, and the multiP11 dæmon may remove it after processing it
and before revealing the session to clients. This way, the PIN could get spread
between different parts of a system — different user accounts, for example.

Structuring Library Isolation
-----------------------------

It may sound offending to efficiency to run each PKCS \#11 backend library in a
separate process. And if not designed with care for efficiency, then it might be
— although the security gains may easily defend this to still be acceptable. But
we can work around inefficiencies.

Keep in mind that the multiP11 is a switchboard for PKCS \#11 operations, and
may benefit from locality of reference. The same applies to the individual
backend libraries, keeping these focussed on a limited task is never going to be
disadvantageous in terms of cache performance of processors. As for the need to
switch processes, it is good to realise that processes can run in different
cores on any modern CPU, so that classical concerns for reloading MMU pages
don’t necessarily apply either. The one concern of some interest is perhaps the
need to switch to kernel space.

What remains now, is to find a secure-yet-efficient manner of relaying messages
between the independent processes. For “secure”, we need to avoid that backend
libraries can address each others’ resources; for “efficient”, we aim to avoid
as much as possible of the processing of data. Since multi-core architectures
operate on shared memory, any solution that shares memory is ideal.

### Sharing Address Space

When child processes are created with `fork()` or `clone()`, the page tables are
copied and (conceptually) the contents of memory are cloned. This means that the
data is shared up to the moment of forking, but that address space continues to
be shared; so we could pass around data, have it stored in another process and
index it with pre-parsed Quick DER structures, or pointer addresses in PKCS \#11
calls.

### Sharing Memory Regions

There are a few schemes where memory regions are shared; these may combine with
sharing address space; otherwise, an address offset must be applied to any
addresses passed between two processes. Additionally, the data structures passed
into a PKCS \#11 call can be filled by the call, and used directly, or after
applying another address offset.

One mechanism for sharing memory is `shmget()` which has the disadvantage that
the parties involved need to share the same user and/or group to be able to
share the memory. This may be problematic if the dæmon connects to multiple
backends, and needs to have a different face towards each of them. Running as
`root` conflicts with the requirements, other privileges may make this possible;
however, `setgid()` can always be called with a group into which a user belongs,
so the multiP11 dæmon could share a separate group with each of its backend
libraries and access could be granted solely on group identity (and the shared
memory region would be owned by multiP11 or the backend library). The need to do
the extra kernel calls for every message exchange may be more costly than
copying memory.

An alternative could be `msgget()`, which at least has separate ownership for
reading and writing a message queue — which could be setup once and then used
forevermore. This might pass a message, possibly including addresses in fixed
locations; the addresses might need an offset or it might always be allocated in
a shared address space (allocated prior to forking). There would be a separate
message queue for sending and receiving.

Finally, there is `mmap()` which can be used to map a file (or none, if so
desired) into a memory address space. Memory mapping is a way of sharing memory,
and file access privileges apply at the time of mounting but not at the time of
reading and writing; this means that tricks with `setgid()` are not as costly as
with the `shmget()` solution. Interesting about this solution is that a `fork()`
would replicate the memory map at the given location, and control would still be
with the same program as the parent, so it is possible to `munmap()` memory
regions that are not considered available to the backend being forked. This may
be done in preparation of loading the backend library, which is just what we
need because we consider the backend library to be potentially rogue. In this
model, we also do not need to have multiple user identities; we just use many
`mmap()` calls in the parent and in child processes we `munmap()` most of them.
When no files are backing the mapped memory, then there is no way for the child
to regain access to the previously mapped memory.

### Synchronisation Model

Given a shared memory space, a number of options arise for inter-process
synchronisation. What we desire is a sort of coroutine call initiated by the
multiP11 dæmon and executed by a given backend. Note that PKCS \#11 calls
usually return immediately

The simplest method to doing this in the backend is probably to have a thread
pool listen to a message queue, over which requests arrive. When they do, they
are used to construct a PKCS \#11 call which is passed down to the backend
library. The thread pool can have a limited size if we assume that no blocking
calls are ever made. This means that the only blocking call in PKCS \#11, namely
`C_WaitForSlotEvent()`, will be made non-blocking (and wrapped, if so desired)
in either the backend or the multiP11 dæmon. Since this may be shared with a
future extension for event notification, the ideal place for doing this would be
the multiP11 dæmon.

A message queue needs a locked counter to indicate that the next message may be
picked up. This can be implemented with any mechanism that is purely
memory-based. POSIX Threads have a facility of this nature, and a `futex()`
could also work when the platform is Linux.

As for the return of a call result, this may take another mutex to deliver the
response. The multiP11 dæmon probably needs at least one thread to monitor the
outcome of each backend, but it may also choose to spark a new thread for any
incoming request, and have that wait on the request at hand. The latter has the
advantage of storing processing data in the thread’s stack, which is helpful
with more complex matters such as a `C_FindObjects()` call that moves from one
backend to the next according to the backend list for a given client.

To keep client requests sequenced, there may be a thread for each client that
made a proper connection. Or, to help fight off denial-of-service attempts, once
a client has succeeded authentication it might get its own Thread assigned. That
would simplify keeping global state such as counting the rate of attempted
connections and perhaps even seeing the same IP popup repeatedly. (To that end,
a Bloom filter with various segments could be used — every time a connection is
made from an IP that is in a few of the segments already, also add it to other
segments, and reject when all segments know about it already. Reset filter
segments at a fixed pace. Needless to say that any identying information could
work here, not just an IP. But the point is that it is helpful to keep the
pre-authentication phase shared.)

### Sharing Stack Regions

Next to considerations of sharing data memory regions, there would be a facility
for sharing stack regions as well. This might help to prepare PKCS \#11 function
calls and apply them to prepared arguments from a varargs list, for instance. It
is common for assembler programs to reference a fixed offset to function
arguments, and though these are normally stack-allocated the same code (except
perhaps a different function entry point) could be used on variable arguments at
some location in data memory. There is no practice in doing this for C functions
however, let alone having it done in a portable manner. It is probably better to
actually copy the arguments one by one.

Given this, we come to a conclusion that seems to make sense in general, and
that is the exchange of preparsed Quick DER entries between the multiP11 dæmon
and the processes that host each backend library. Building up the PKCS \#11
function call needs to be done only once, when the further processing in the
multiP11 dæmon is founded on Quick DER items; and given that these are
straightforward `<address,length>` tuples that represent elementary bits of
data, each bit stored in its own structure field, this ought to be quite doable.
Any repeatedly derived content (such as an integer tag value) can be passed
around in an additional location, either as call arguments or stored in a
surrounding structure around the Quick DER structure. It should be trivial for a
backend library process to pick and choose the right pieces for the call at
hand.

### Selection of Isolation/Sharing Mechanism

Based on the observations above, we choose to `mmap()` a section for each
backend library, using `MAP_ANONYMOUS|MAP_SHARED` as flags; then to `fork()` a
backend process for each of the libraries, which start to `munmap()` every
mapped bit except for the one(s) relating to their own purpose. Memory
management will be specific to the backend library and will aim to recycle the
memory used inasfar as possible. We will assume that any swap memory will be
encrypted. This has the following attractive properties:

-   portable to POSIX platforms (Windows is not suitable for this kind of
    application)

-   share memory region and address space between multiP11 dæamon and backend
    library

-   isolate the memory region between backend libraries

-   isolate the multiP11 dæmon’s address space from that of the backend
    libraries

The latter works better if we make the split into backends as soon as we can; we
will not need to close as many resources if we do it that way.

The memory shared can become a limiting factor, though one could argue that it
is in the interest of security to be unable to grow without bounds. If it is
desirable to at least grow a bit, then we might consider forking additional
backends for libraries that are already opened, and assign it with a larger name
space. All new connections go to the new library; this would complicate future
support of event notifications, but not to an unworkable degree. This is
especially useful when properly authenticated clients arrive in a larger number
than predicted; a maximum number may be set per backend process, and new
instances of such backends may be created as the need arises. The advantage of
doing this early is then lost, so a more advanced resource-freeing facility will
then be needed (but the cleanup process of the multiP11 dæmon itself could prove
useful for that — when called with a modifying argument such as a backend
instance reference).

Who does PKCS \#11 NAT?
-----------------------

If we trusted the backends, we could let them pass object handles, but it is
probably better if we didn’t.  On the other hand, we communicate with Quick DER
fragments and might need to reallocate the space for the object handles when
their size changes.

Interestingly however, we will allocate the space in which our PKCS \#11
backends may write.  This means that we will offer the room for these values
(and object handle is stored in an `ACK-OBJECT-HANDLE`, which is an alias for
`ACK-ULONG`, which in turn needs a known space of up to 5 bytes.  The value to
return is allocated by the main dæmon.

In fact, there is an option where the return space is allocated in pre-DER
format, where the main dæmon packages it for shipping with Quick DER.  Or, when
it is used as a library, it might simply copy the data.  If the backend is a
Remote PKCS \#11 node however, it may be more efficient to pass back Quick DER
(though addresses cannot be mapped in that case).

Hmm, choices, choices...

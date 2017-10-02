Remote PKCS \#11
================

>   *ARPA2 software is built around single signon in Kerberos. It publishes
>   public-key credentials in LDAP under one’s domain name. The one thing to add
>   is a private-key infrastructure, which we build around PKCS \#11. We
>   specifically use remote PKCS \#11.*

Reasons for using remote PKCS \#11
----------------------------------

-   Credentials for remote access already assume online access

-   The same credentials are available everywhere

-   Credentials are not carried around on mobile devices (especially for “soft
    tokens”)

-   Identity infrastructure can centrally rollover keys and manage
    identities/pseudonyms

-   Administrative staff can access/control/retract keying facilities

-   Administrative staff can grant/revoke access to PKCS \#11 objects

-   User choice where to host — on a controlled Raspberry Pi or at a
    smarter-than-me ISP

-   TLS Pool is an example application that greatly benefits from this setup

-   FireFox could access remotely/separately managed client credentials

How to access remote PKCS \#11
------------------------------

-   Use LDAP references in `pkcs11PrivateKey` to locate PKCS \#11 locations

-   For a `pkcsPrivateKeyRemote`, find connection details in LDAP

-   Fetch a service ticket for `pkcs11/remote.host.name@PKCS11.ISP.REALM`

-   Over a secure connection, GSS-RPC
    [[RFC2203](<https://tools.ietf.org/html/rfc2203>)] for RPC
    [[RFC5531](<https://tools.ietf.org/html/rfc5531>)] and XDR
    [[RFC4506](<https://tools.ietf.org/html/rfc4506>)]

    -   We would not really use RPC but would be bothered by its program number
        stuff

    -   Map function and in(out)-params to XDR, map XDR back to retval and
        out-params

    -   Use GSS-API's wrap and unwrap functions for encryption and decryption

    -   For TCP, add a size before each packet; always add stuff like request ID

-   Kerberos SSO inherits from the client’s context, accessible to PKCS \#11
    libraries

-   Secure the operation, return codes, parameters in/out

-   Use GSS-API client identity to select a number of available PKCS \#11 object
    sets

-   PKCS \#11 implements object sets with overlap, each with a unique token
    serial number

-   Present these object sets as though they were different tokens in different
    slots

-   User may assign a label to each of those “tokens” or layered object sets

-   Each token has a separate User PIN (ignored) and SO PIN (or
    `CKF_SO_PIN_LOCKED`)

-   Access control dictates visibility of tokens that were added to a certain
    object set

-   Management software may work on these object sets

-   User selects a particular token or object set and logs in with the
    corresponding PIN

Object sets and Tokens
----------------------

-   An object set is a set of PKCS \#11 objects with internal index numbers

-   A token is a combination of a number of object sets, as though they were
    layers

-   Authenticated GSS-API users are granted access at the token level (rdonly,
    rdwr, cow)

-   Users compose tokens with a label and PIN, and they get a random serial
    number

    -   Every GSSRPC connection can have a token without `CKF_TOKEN_INITIALIZED`

    -   Actually, multiple server-side backends can be shown as multiple of
        these tokens

    -   This token has a GSSRPC-reserved, random `serialNumber` (96 bits
        entropy)

    -   `C_InitToken()` can setup that token, assign a label, and create a next
        new slot

    -   During initialization, an empty layer 0 is created and the GSS-API user
        may rdwr

    -   It may be useful to store the SO-PIN, to distinguish SO and User within
        GSS-API

    -   User PIN operations are silently ignored, since GSS-API credentials are
        used

-   Tokens offer `CKF_PROTECTED_AUTHENTICATION_PATH` and accept/ignore user PINs

-   Writes to a token ends up in its own layer 0 (which may be incorporated
    elsewhere)

-   So: layers come down to “objects created in token XXX”

-   Object handles \<token layer number, object set index\> are quick to
    dispatch

-   Implementation question: Can sessions immediately pickup on changes in
    another?

    -   Nice to have, as it supports a generating app independently from a using
        app

    -   Nice to have, as it means that LDAP-stored private keys are immediately
        usable

    -   It is not promised in PKCS \#11, but may nonetheless be possible

    -   Probably need to hold objects that were published in LDAP at/before
        session start

    -   Rollover could use garbage collection based on LDAP-entries and open
        sessions

-   Implementation question: Can one layer be dispatched to downstream PKCS
    \#11?

    -   Nice to have, as it makes hardware modules accessible

    -   Needs to split object handles — translation tables being a
        simple/general approach

    -   May disable cow, unless Wrap/Unwrap are being settled between layers

    -   How to produce (where to store) the User PIN and SO PIN for the
        downstream token?

        -   Perhaps a locally stored value, encrypted with the upstream User PIN
            / SO PIN

        -   Note that multiple clients, each with their own User PIN, can access
            one backend

Management by Administrative Staff
----------------------------------

To manage identities in a user’s PKCS \#11 token, administrative staff must be
permitted to login and make changes. There are two ways to do this.

The administrative software may take on the identity of the token user, through
such mechanisms as S4U2Proxy with Constrained Delegation (if that is
sufficiently secure, which may be true if this is all in-house). This is
possible when the user addresses a service that delegates to such management —
this could be used for such things as creation of new user identities at the
user’s request over some other protocol intended for management.

But there are also administrative tasks, such as key rollover and cleanup. These
are needed for fully automated identity management, but the user should not have
to take initiatives for routine management. The PKCS \#11 API defines the SO PIN
for such purposes, but `CKU_SO` and `CKU_USER` cannot coexist, rendering the SO
an unsuitable party for key creation and rollover.

The last option is to make use of the layers that combine to form a remote PKCS
\#11 token. This could be used to have a “managed layer” that is only read by
the client, and written by the management application which logs in as its user.
If the layer is implemented with a backend PKCS \#11 API, there will be a need
to login, which is doable through an extra session. This will however introduce
a need for decryption of the stored User PIN by the management software, and not
just on behalf of an authenticated remote PKCS \#11 protocol client. The SO PIN
can be given the same treatment, for the purpose of clearing the PKCS \#11 token
when needed; there probably will not be a need, ever, to re-initialise the User
PIN because that is encrypted to the client.

 
-

 

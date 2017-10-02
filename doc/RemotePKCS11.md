# ASN.1 messaging format for PKCS #11

> *Having an ASN.1 model for PKCS #11 calls and returns is useful for remote
> access to PKCS #11 repositories.  The transport can be handled orthogonally,
> and the issues of encoding, encryption and authentication can be resolved at
> that layer too, though their use may be put to use in PKCS #11 as well, for
> instance to supply a user's identities.  This document limits itself to the
> method of mapping (current and future) PKCS #11 versions to ASN.1 notation.*


	RemotePKCS11 MODULE ::= BEGIN



## Primtive Data Types

  * Map `CK_BYTE` to `OCTETSTRING(SIZE(1))`.
  * Map `CK_CHAR` to an `IA5STRING`, possibly with `SIZE()` constraints.
  * Map `CK_UTF8CHAR` to an `UTF8STRING`, possibly with `SIZE()` constraints.
  * Map `CK_BBOOL` to a `BOOLEAN`; map values `CK_TRUE` to `TRUE` and `CK_FALSE` to `FALSE`.
  * Map `CK_ULONG` to `INTEGER (0..4294967295)`
  * Map `CK_LONG` to `INTEGER (-2147483648..2147483647)`
  * Map `CK_FLAGS` to `BIT STRING`


## Array Types

Some of the primitive data types are described in ASN.1 as a size-constrained
form of a repetitive structure; arrays of such types simply expand the
repetitive structure by lifting the size constraint.  This applies to
`CK_BYTE`, `CK_CHAR`, `CK_UTF8CHAR`.

Other array types are described with pointer and length in the PKCS #11 API,
and are treated as described for pointers.  They may be input and/or output
parameters, where PKCS #11 constraints will be used to shield the user program
from too much content.

TODO: Ship the size of an output parameter?  Makes the backend calll look
more like the one the user sent.


## Principles of Mapping

  * This specification defines a reversible mapping from PKCS #11 data types
    to ASN.1 structures; where necessary, it additionally defines how data
    is reversibly translated from PKCS #11 data to ASN.1 data; but it does
    not replicate the resulting constant value definitions in terms of ASN.1
    because that is only a derived form of what is specified here.
  * PKCS #11 functions have parameters, which are passed back and forth.
  * Do not map `_PTR` types to ASN.1 but instead dereference the information
    and pack it into ASN.1 structures.  The pointer may however be used to
    couple the information between a call and return data structure.
  * Do not map `CK_NULL` to ASN.1 but instead make all the places where it
    might occur `OPTIONAL`, prefixed with an explicit contextual tag.
    **CHANGED:** pointers can be sent as `null [0] NULL`, meaning the `NULL_PTR`
    was supplied, as `nodata [1] NULL` meaning that a valid pointer was supplied
    but there is no use in passing the information, or as the contents with
    a label, the `[2]` tag and the referenced content type.  Note that `VOID_PTR`
    is passed as `ANY`, meaning that the content requires further parsing.
    The three variations are combined into a `CHOICE`.  Note that `[1]` is mostly
    an optimisation of `[2]` by not sending old buffer data, but that may have
    security advantages as well when the application assumes that the old buffer
    contents will stay within a process.  Therefore, it SHOULD be used when no
    data is needed to be sent.  Note that lengths that are queried from PKCS #11
    are often passed as an `ULONG_PTR`, and the mechanism works for that as
    well; in those cases, it would send the old (maximum) value to the remote
    PKCS #11 implementation and receive the improved value back.  In cases where
    the data ships as `[2]` and there is an additional field to describe the
    length or size, both are supplied to the remote PKCS #11 implementation, but
    the receiving party *may* implement constraints that signal conflicts between
    these values as constructed by the sending party.
  * Where PKCS #11 describes default values for parameters that may be
    set to `CK_NULL` or another empty form, specify these with `DEFAULT`.
    **Changed:** For reasons of simplicity and consistency with the remote
    PKCS #11 semantics, we now leave these issues to the remote PKCS #11 library.
    This means that we actually send a `NULL_PTR` value and have it interpreted
    on the other end as the library deems useful.
  * Do not map pointer/length combinations literally.  Instead, pass the
    data pointed at and rely on ASN.1 to encode the length.  Note that the
    form in which lengths occur in PKCS #11 includes counters, such as a
    number of attributes in an array.
  * PKCS #11 defines a data type mapping for machine environments; for ASN.1
    other choices can be better.  This is why elementary data types are
    specifically mapped.


## Function Pointers

  * PKCS #11 defines a function pointer list, and there are function pointers
    for locking.  All these will be wrapped by a remote PKCS #11 implementation,
    and where needed they will emit the format described here.
    What remains as an exchange format for a function is a boolean that indicates
    whether the function is provided.
  * `CK_FUNCTION_LIST_PTR_PTR` as used in `C_GetFunctionList` is defined as a
    `BIT STRING` with bits set to 1 for existing functions; this makes this
    message format portable to future versions of PKCS #11:

	CK_FUNCTION_LIST ::= SEQUENCE {
		version CK_VERSION,
		functionlist BIT STRING
	}

## Mapping Function Calls

The general type for a function call is

	FunctionCall ::= SEQUENCE {
		funlistIndex INTEGER,
		parameters SEQUENCE OF FunctionParameter
	}

All the parameters are serialised and appended after this.

	FunctionReturn ::= SEQUENCE {
		returnValue CK_RV,
		parameters SEQUENCE OF FunctionParameter
	}

The function parameters are only sent when they are considered useful in that
direction; so in `FunctionCall`, parameters considered input and inout; and in
`FunctionReturn`, parameters considered output and inout.  A parameter is
considered input when PKCS #11 describes its value upon calling; a parameter
is considered output when PKCS #11 describes its value upon return.

Not all parameters are mapped; a pointer/length combination for instance,
is sent as one structure, even if the length is itself pointed at.  The
FunctionParameter describes a parameter, but MUST skip any lengths that
will be implied by the ASN.1 encoding.  In addition, parameters MAY be
skipped if they need not pass any data.

	FunctionParameter ::= CHOICE {
		null [0] NULL,
		nodata [1] NULL,
		value [2] ANY
	}

**NOTE:** This doubles the idea of the `_PTR` type.  Are we combining two notions here?  Note that absence is a property of parameter in transit, while the NULL value is a property of a pointer.  Also note that nested `CHOICE` is probably permitted when tags differ.  Finally, note that we can easily define types

NULLPOINTER ::= [0] IMPLICIT NULL
ABSENT ::= [1] IMPLICIT NULL

and use them like

	FunctionParameter ::= CHOICE {
		nodata ABSENT
		value ANY
	}

which works better than `value ANY OPTIONAL` could as `SEQUENCE OF FunctionParameter`
makes it impossible to distinguish parameters that have been skipped.

Based on this structure we can fill in the `ANY` value with a pointer, perhaps

	ULONG_PTR ::= CHOICE {
		null NULLPOINTER,
		intval INTEGER
	}

One reason for simplifying by making `nodata` part of `_PTR` structures is that it
makes little or no sense for primitive values, and complex types are always
described with `_PTR` types in PKCS #11.

TODO: This is a meta-type approach; would it be better to work out concrete types for each function?  Probably not; a specification with `[0]` before each concrete parameter and `, ...` at the end requires message format specification updates.

## Transport Bindings

It may be useful to represent contextual information as part of the interface
to the transport layer.  This might reveal such things as the identity of the
user, the identity for the addressed PKCS #11 store, and perhaps secrets
derived from the transport session or created for the given <user,pkcs11>
relationship.

	ContextualInformation ::= SEQUENCE {
		transportProtocol OBJECT IDENTITY,
		relationSecret OCTET STRING OPTIONAL,
		sessionSecret OCTET STRING OPTIONAL,
		userIdentity OCTET STRING OPTIONAL,
		pkcs11Identity OCTET STRING OPTIONAL
	}

Transport connections may support asynchronous use of PKCS #11, which means
that they need a method to bind a request and responses.  This can use the
requestID field in the following constructs:

	TransportFunctionCall ::= SEQUENCE {
		transportID OCTET STRING OPTIONAL,
		requestID OCTET STRING OPTIONAL,
		payload FunctionCall
	}

	TransportFunctionReturn ::= SEQUENCE {
		sessionID OCTET STRING OPTIONAL,
		requestID OCTET STRING OPTIONAL,
		payload FunctionResponse
	}


## Final Remarks

This ends the remote PKCS #11 specification.

	END

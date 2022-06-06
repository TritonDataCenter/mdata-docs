# Triton SmartOS Metadata Protocol Specification (Version 2)

## 1. Introduction

Triton [SmartOS][1] provides a facility to store arbitrary key-value pairs
alongside the configuration of a virtual guest instance.  In the global zone of
a standalone SmartOS system (in use as a hypervisor), the root user may inspect
or update metadata values using the [`vmadm(8)`][4] tool.  In a
[Triton][3] cloud deployment, the customer or administrator may use
[CloudAPI][2] to query and update metadata for guest instances.

Once metadata has been provided to the system, it can then be accessed from
within the guest instance (either an OS virtualised, or zone, instance; or a
KVM hardware virtualised instance).  It is strongly recommended the use of [the
provided command-line tools][5] when accessing the metadata services.  These
tools currently build on SmartOS and GNU/Linux, with support for more platforms
to come.  If you are a Triton Data Center user, or are using the SmartOS
template images available at [images.smartos.org][6], then you may already
have the `mdata-client` tools installed in your guest instance.

## 2. Conventions

Some numeric values are represented in hexadecimal. They are typeset with a
`0x` prefix -- e.g. `0x1F` is equivalent to the decimal value _31_.

Some protocol wire data is represented directly in this document as strings as
defined by the C Programming Language.  These are generally represented in a
`"monospace font"`, surrounded by double quotes.  For example, a linefeed
character (a single byte value of `0xA`) would be represented as `"\n"`.

## 3. Operating System-specific Concerns

### 3.1. Serial Ports

In the case of a client implementation using the emulated serial port to
communicate with the hypervisor (e.g. in a KVM hardware-virtualised machine),
it is advisable to read (and discard) any ingress bytes pending on the serial
port when beginning a transaction.  Depending on your guest operating system,
and on the unknowable state of the hypervisor on the remote end of the serial
link, you may be required to leave a pause (say, 100 milliseconds) to ensure no
more data will arrive.  This flush procedure attempts to ensure a clean slate
for sending new requests to the metadata service.

Once all ingress data is discarded from the serial port, you may send a single
linefeed (`"\n"`) through the connection.  The expected response from a
conformant Version 1 or Version 2 host is a single line containing the string
"invalid command":

    send:      "\n"

    receive:   "invalid command\n"

Additionally, while communicating via the serial port you should hold an
exclusive lock over the serial port.  If your platform does not support
exclusive locks, you should use whatever advisory locking mechanism will be
used by other tools which will consume metadata via the serial port.  The
[provided tooling][5] do this on supported platforms -- e.g. by using
the fcntl(2) locking mechanism on UNIX platforms.

## 4. Transport

The wire protocol implemented in the metadata agent service is designed to be
easily implementable on a range of programming platforms, including C and
Javascript.  As the protocol is used in both OS-virtualised and hardware
virtualised instances, the transport layer of the protocol is designed to work
over both a reliable domain socket connection and an emulated serial link.

### 4.1. Negotiation

This document describes the second version of the metadata protocol.  This
version of the protocol begins with a negotiation sequence to determine if it
is supported by the hypervisor.  The negotiation sequence is framed as a series
of Version 1 style requests and responses that will only succeed on a host that
supports Version 2.  To begin negotiation, send the Version 2 negotiation
request to the server:

    send:      "NEGOTIATE V2\n"

    receive:   "V2_OK\n"

If you receive a string which is _not_ `"V2_OK\n"`, then the host likely does
not support this version of the protocol and this document does not apply.
Otherwise, you may consider Version 2 support available and begin sending
requests as Version 2 protocol frames.

Once you have negotiated Version 2 protocol support with the hypervisor, all
future communication should be formatted as Version 2 protocol frames.  This
frame format allows for various messages to be passed back and forth between
the guest and hypervisor without reinventing basic integrity features for each
message type.

### 4.2 Frame Format

A valid frame begins with the string `"V2 "` -- note the single trailing space.
Immediately following this space is a set of fields, described in the table
below, that should be separated with a single space.  The allowed field values
are defined such that they do not ever contain spaces.  Each frame is
terminated with a single ASCII linefeed (`\n`).

|| **Field** || **Description** || **Format** ||
|| _Body&nbsp;Length_ || The length (in bytes) of the _Body_ of the message. || A variable-length, ASCII representation of the decimal digits that make up the length. ||
|| _Body&nbsp;Checksum_ || The CRC32 checksum of the _Body_ of the message. || A zero-filled, eight-character, lower-case ASCII representation of the hexadecimal digits that make up the CRC32 value. (e.g. `000ae1d8`) ||
|| _Body_ || The active content in the message. || Described in _&sect;4.3 Frame Body_. ||

The [CRC32 checksum][7] is calculated using the polynomial `0xEDB88320`.  The
implementation of this checksum function [included in the command-line
tools][8] is considered normative for the purposes of this document.

### 4.3 Frame Body

The _Body_ field is further sub-divided into a series of sub-fields.  These
sub-fields are also each separated by a single space, and are defined such that
their values do not contain spaces.  In the case of an optional value, neither
the field value nor its separating space, are present in the string.

The primary reason for defining the _Body_ as a set of sub-fields is to clearly
denote the contiguous block of bytes that are used to calculate the _Body
Length_ and _Body Checksum_.  Note that the trailing linefeed on the end of the
frame is _not_ included in the _Body_ itself, and should not be included in
integrity checking.

|| **Body Field** || **Description** || **Format** ||
|| _Request&nbsp;ID_ || A unique identifier for this request to be used in matching Requests with their respective Responses. || A zero-filled, eight-character, lower-case ASCII representation of the hexadecimal digits of a randomly generated 32-bit number.  (e.g. `dc4fae17`). ||
|| _Code_ || The code denoting the requested operation in a Request, or result of the requested operation in a Response -- i.e. generally Success or some Error condition. || A single word, generally in upper-case, using only ASCII characters and not including any spaces.  e.g. `SUCCESS`, `NOTFOUND`, etc ||
|| _Payload_ || The input parameter to a Request, or the returned value in a Response. || This field is optional.  It is an arbitrary set of contiguous bytes, and must be BASE64-encoded so that its representation does not contain any invalid characters.  The meaning of the decoded value is specific to the Request or Response message type. ||

### 4.4. Diagrammatic Example

The following is a simple example of a Response to a `GET` Request.  The
_Payload_ value is `"[]"`.

                                              ,- Body (CRC32/Length is of
                    _____________________ <--/         this string)
     V2 21 265ae1d8 dc4fae17 SUCCESS W10=\n
     ^  ^  ^        ^        ^       ^   ^
     |  |  |        |        |       |   \--- Terminating Linefeed
     |  |  |        |        |       \------- Payload
     |  |  |        |        \--------------- Code
     |  |  |        \------------------------ Request ID
     |  |  \--------------------------------- Body Checksum (CRC32)
     |  \------------------------------------ Body Length
     \--------------------------------------- So that this can be a V1 command
                                              as well, we start with "V2".
                                              This will be a FAILURE on a
                                              host that only supports the
                                              V1 protocol.

### 4.5. Notes

The entire protocol 'frame' is, itself, a valid Version 1 command.  This is to
ensure that even if the negotiation procedure is not completed correctly, and a
Version 2 is falsely assumed, older hypervisors will at least generate _some_
response.  That response cannot be depended on, other than it will likely be
one of the following strings (including a terminating linefeed):

        "FAILURE\n"
          or
        "invalid command\n"

## 5. Operations

The metadata protocol allows a variety of operations to be performed, either
querying or modifying the metadata store for the particular guest instance.
Each well-formed request frame from the guest will result in a single
well-formed response frame from the hypervisor, bearing a matching _Request ID_
field.

### 5.1. Common Errors

It is always possible for a request to fail.  The client is responsible for
ensuring the _Response Code_ field is of the expected form before interpreting
the _Payload_ value.  Some error codes may be returned as a Response to any
Request:

|| **Response Code** || **Condition/Payload Format** ||
|| `FAILURE` || Some kind of error occurred during processing.  If the payload is present, it will contain an error string that should be displayed to the operator. ||

### 5.2. GET

A `GET` request, as used by the `mdata-get(8)` command-line tool, allows the
consumer to fetch the value of the named key-value pair from the metadata
store.

#### 5.2.1. Request

The request payload for a `GET` request is simply the name of the requested
metadata value.

#### 5.2.2. Response

|| **Response Code** || **Condition/Payload Format** ||
|| `SUCCESS` || The payload is the value of the requested key-value pair. ||
|| `NOTFOUND` || The requested key-value pair was not found in the metadata store.  The payload is not present. ||

### 5.3. KEYS

A `KEYS` request, as used by the `mdata-list(8)` command-line tool, results
in a list of all custom key-value pairs in the metadata store.

Note that in current implementations of SmartOS and Triton, this list
does _not_ include the keys in the `sdc:` namespace.
<!-- XXX TODO Document where the sdc: namespace keys come from. -->

#### 5.3.1. Request

This request requires no inputs, and thus does not require a payload.

#### 5.3.2. Response

<!-- markdownlint-disable MD033 -->
|| **Response Code** || **Condition/Payload Format** ||
|| `SUCCESS` || The payload is a linefeed-separated list of the names of the available custom metadata keys, if any exist.<br><br>e.g. `"root_authorized_keys\nuser-script\n"` would correspond to the set: `root_authorized_keys`, `user-script`. ||
<!-- markdownlint-enable MD033 -->

### 5.4. PUT

A `PUT` request, as used by the `mdata-put(8)` command-line tool, creates a
new key-value pair in the metadata store.  If the key to create already exists,
the value of the existing pair is replaced with the provided value.

Note that in current implementations of SmartOS and Triton, the user
may not update keys in the `sdc:` namespace -- these are considered read-only
to the guest instance.

Additionally, the metadata store as visible through CloudAPI is only eventually
consistent with the metadata store as visible to the guest instance through the
metadata protocol.  Some non-trivial latency for `PUT` values will be
measurable if your system requires polling for outbound metadata values.

#### 5.4.1. Request

A `PUT` request requires _two_ input arguments, each of which may potentially
contain spaces -- the _key_ name of the intended key-value pair, and the
intended _value_.  Each of the _key_ and _value_ are separately BASE64-encoded,
and separated by a single space.  This concatenated string is then, itself,
BASE64-encoded as part of the regular framing process.

#### 5.4.2. Response

|| **Response Code** || **Condition/Payload Format** ||
|| `SUCCESS` || The metadata store was updated as requested.  The payload is not present. ||

### 5.5. DELETE

A `DELETE` request, as used by the `mdata-delete(8)` command-line tool,
removes a particular key-value pair from the metadata store.  This operation is
successful even when the nominated key-value pair did not exist.

#### 5.5.1. Request

The request payload for a `DELETE` request is the name of the metadata
key-value pair to delete.

#### 5.5.2. Response

|| **Response Code** || **Condition/Payload Format** ||
|| `SUCCESS`       || The requested key-value pair was removed from the metadata store, or did not originally exist.  The payload is not present. ||

<!-- Links/References -->

[1]: http://www.smartos.org/

[2]: https://api.tritondatacenter.com/docs/public/index.html

[3]: http://www.tritondatacenter.com/products/private-cloud

[4]: https://github.com/tritondatacenter/smartos-live/blob/master/src/vm/man/vmadm.8.md

[5]: https://github.com/tritondatacenter/mdata-client

[6]: https://images.smartos.org

[7]: http://en.wikipedia.org/wiki/Cyclic_redundancy_check

[8]: https://github.com/tritondatacenter/mdata-client/blob/master/crc32.c

# Joyent Metadata Protocol Specification (Version 2)

## Introduction

Joyent [SmartOS][1] provides a facility to store arbitrary key-value pairs
alongside the configuration of a virtual guest instance.  On a SmartOS system,
the root user may inspect or update metadata values using the [vmadm\(1M)][4]
tool.  In a [SmartDataCenter][3] cloud deployment, or in the Joyent Public
Cloud, the customer or administrator may use [CloudAPI][2] to query and update
metadata for guest instances.

Once metadata has been provided to the system, it can then be accessed from
within the guest instance (either an OS virtualised, or zone, instance; or a
KVM hardware virtualised instance).  Joyent strongly recommends the use of [the
provided command-line tools][5] when accessing the metadata services.  These
tools currently build on SmartOS and GNU/Linux, with support for more platforms
to come.  If you are a Joyent Public Cloud customer, or are using the SmartOS
template images available at [images.joyent.com][6], then you may already
have the mdata-client tools installed in your guest instance.

## Protocol Overview

The wire protocol implemented in the metadata agent service is designed to be
easily implementable on a range of programming platforms, including C and
Javascript.  As the protocol is used in both OS-virtualised and hardware
virtualised instances, the transport layer of the protocol is designed to
work over both a reliable domain socket connection and a serial link.

### Negotiation

This document describes the second version of the metadata protocol.  This
version of the protocol begins with a negotiation sequence to determine if it
is supported by the hypervisor.  The negotiation sequence is framed as a series
of Version 1 style requests and responses that will only succeed on a host that
supports Version 2.

In the case of a client implementation using the emulated serial port to
communicate with the hypervisor, it is advisable to read (and discard) any
ingress bytes pending on the serial port when beginning a transaction.
Depending on your guest operating system, you may be required to leave a pause
(say, 100 milliseconds) to ensure no more data will arrive.  This attempts to
ensure a clean slate for sending new requests to the metadata service.

Once all ingress data is discarded, send a single line-feed (byte value 0xA)
through the connection.  The expected response from a conformant Version 1
or Version 2 host is a single line containing the string "invalid command":

    send:      \n

    receive:   invalid command\n

Once you receive this error, you should proceed to send the Version 2 negotiation
request to the server.

    send:      NEGOTIATE V2\n

    receive:   V2_OK\n

If you receive a string which is _not_ `V2_OK`, then the host likely does not
support this version of the protocol and this document does not apply.
Otherwise, you may consider Version 2 support available and begin sending
encapsulated `V2` requests.

### Transport

Once you have negotiated Version 2 protocol support with the hypervisor,
all future communication should be formatted using 


### Request/Repsonse

#### GET

#### KEYS

#### PUT

#### DELETE




<!-- Links/References -->

[1]: http://www.smartos.org/

[2]: https://api.joyentcloud.com/docs/public/index.html

[3]: http://www.joyent.com/products/private-cloud

[4]: https://github.com/joyent/smartos-live/blob/master/src/vm/man/vmadm.1m.md

[5]: https://github.com/joyent/mdata-client

[6]: https://images.joyent.com

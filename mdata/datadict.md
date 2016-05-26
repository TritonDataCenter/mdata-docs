# Joyent Metadata Data Dictionary

## Introduction

Joyent [SmartOS][1] provides a facility to store arbitrary key-value pairs
alongside the configuration of a virtual guest instance.  The contents of this
metadata store are available to the guest instance, either through [the
command-line tools][2], or the directly via the metadata protocol itself.  The
protocol is described in a related document: [Joyent Metadata Protocol
Specification (Version 2)][6].

This document describes various well-known key names that have defined
semantics.  Some of these keys are expected (or required) to be used in
particular ways during the boot process of a guest instance; they form an
interface contract between the guest instance itself, the provisioning system
and the consumer.  This interface definition allows us to provide a consistent,
predictable user experience to consumers, and to enable advanced functionality
like zero-touch custom image management.

## Read-only Metadata (The `sdc:` Namespace)

The SmartOS hypervisor provides a set of read-only keys to the guest instance.
The names of these keys are prefixed with the string `sdc:`.  They provide
various information about the host system and the guest instance.  The names of
these keys do _not_ appear in the output of the `KEYS` metadata operation, or
the output of the _mdata-list\(1M)_ command.  They may be read with the `GET`
operation, but not modified with `PUT` or `DELETE`.

### `sdc:uuid`

Every guest instance in a SmartOS hypervisor, and by extension in a
[Triton][4] cloud, has a [universally unique identifier][5] or UUID
that identifies it.  This UUID is the identifier used to select the particular
guest for control commands and configuration updates in the provisioning system
API.

### `sdc:server_uuid`

Every SmartOS hypervisor has a UUID that identifies it.  So that consumers can
reason about the placement of their guest instances relative to one another,
the UUID of the hypervisor on which a guest instance is running is made
available.

### `sdc:datacenter_name`

Every [Triton][4] datacenter has a name.  This string is generally
configured by the cloud operator to represent the physical location (and
potentially some fault isolation boundary) of the particular datacenter.  For
example, in the Joyent Public Cloud, there are datacenters with names such as
`us-west-1` and `us-east-1`.

### `sdc:nics`

A VM has a number of associated NICs whose configuration and details are
stored in this key as a JSON array of objects. Each object may have the
following fields:

* `mac` is a string containing the MAC address of the NIC that the described
  configuration applies to. This is always present.
* `interface` is a string containing the name of the interface used by
  [Triton][4]. The operating system should make a best effort to
  assign the NIC the same name. This is always present.
* `ips` is an array of IPv4 and IPv6 addresses that should be assigned to
  the interface. The addresses are written in CIDR notation to indicate
  the prefix length of their subnet. In addition to IP addresses, the
  array can also contain the strings `dhcp` and `addrconf`. If `dhcp` is
  present, then DHCPv4 should be performed on that interface. If `addrconf`
  is present, then the node should respect Router Advertisements received
  on that interface and should assign SLAAC addresses or perform DHCPv6 on
  that interface as appropriate.
* `primary` is a boolean indicating if this is the primary NIC. Only one
  NIC will ever be the primary, and its `gateways` will be used.
* `gateways` contains an array of strings of IPv4 and IPv6 addresses to use
  as the default gateways. This field should only be used if it is the
  primary NIC.
* `mtu` is a number indicating the MTU in bytes that the operating system
  should use for this interface.

### `sdc:resolvers`

When present, this field contains an array of IPv4 and IPv6 addresses to
use for name resolution.

### `sdc:hostname`

When present, this field contains a string containing the hostname for the VM.

### `sdc:dns_domain`

When present, this field contains the domain that should be searched
when performing a host name lookup. This would normally be placed into
`/etc/resolv.conf` after the `search` keyword.

### `sdc:routes`

This field contains a collection of static routes to add to the operating
system's routing table. It is represented as an array of JSON objects,
where each object has the following keys:

* `dst` is a string of the destination address or subnet.
* `gateway` is a string of the gateway address through which the destination
  can be accessed.
* `linklocal` is a boolean indicating if the destination is on-link. When true,
  `gateway` contains one of the IP addresses of the interface through which
  `dst` can be reached.

## Metadata Keys With Defined Guest Behaviour

### `root_authorized_keys`

The value of this key is a set of linefeed-separated SSH public keys, in the
format of an OpenSSH `authorized_keys` file.  In KVM virtual machines, this key
is read during first boot and written to `/root/.ssh/authorized_keys`, or
wherever the SSH keys for the root user are located.  This mechanism allows us
to provide seamless login using the same SSH keys the user has placed within
their account in the provisioning system.

SmartOS instances do not generally make use of this key.  Instead, a more
advanced mechanism called SmartLogin is used -- SmartLogin queries SSH keys
from the provisioning system at each login.

### `user-data`

This key has no defined format, but its value is written to the file
`/var/db/mdata-user-data` on each boot prior to the phase that runs
`user-script`.  This file is _not_ to be executed.  This allows a configuration
file of some kind to be injected into the machine to be consumed by the
`user-script` when it runs.

### `user-script`

This key may contain a program that is written to a file in the filesystem of
the guest on _each_ boot and then executed.  It may be of any format that would
be considered executable in the guest instance.

In a UNIX system, a [shebang line][3] (e.g. `#!/bin/ksh`) may be used to
specify the interpreter for the script.  The script may also _not_ have a
shebang line, at which point the traditional UNIX behaviour of executing the
script with the current shell will occur.  The current shell in this context is
generally `/bin/bash`, or some other Bourne-compatible shell.

### `sdc:operator-script`

This key is set by the provisioning system itself and, if present, represents
a program that _must_ be executed on every boot.  It must be executed prior to
running any `user-script`.  If this script exits with a error status, the
error status is to be ignored and the boot process should continue.

This script hook enables the provisioning system to affect controlled,
arbitrary change in the guest instance.  This facility is critical to
the creation phase of zero-touch custom image management.

## Recommendations For Third Party Vendors

If you are producing an image to be deployed in a Triton instance, or
specifically in the Joyent Public Cloud, you should follow the rules outlined
previously in this document with respect to key names and intended behaviour.
In addition, if you wish to provide additional configuration options that are
specific to your product, you should do so with your own key namespace prefix.

To ensure that there are not inter-vendor key name conflicts, vendors are
encouraged to use the reversed version of their corporate DNS domain as a
prefix.  For example:

    com.ubuntu:user-data
    
        or
    
    com.riverbed:stingray-license-key

<!-- Links/References -->

[1]: http://www.smartos.org/

[2]: https://github.com/joyent/mdata-client

[3]: http://en.wikipedia.org/wiki/Shebang_%28Unix%29

[4]: http://www.joyent.com/products/private-cloud

[5]: http://en.wikipedia.org/wiki/Universally_unique_identifier

[6]: protocol.html

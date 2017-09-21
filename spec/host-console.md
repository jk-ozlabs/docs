OpenBMC console implementation
==============================

Target audience: system deployers & administrators, OpenBMC developers

Overview
--------

The OpenBMC console infrastructure provides a bi-directional channel for serial
data between the host system and an external entity.

The host-side of the console channel is typically a UART-like interface. With
current hardware supported by OpenBMC, this will appear as a 16550a serial
device. This is well supported by common operating system implementations.

External interfaces
-------------------

There are a number of methods for an external entity (user or automated system)
to access the host console:

 - through secure shell (ssh) access on a dedicated port;

 - through an IPMI Serial over LAN (SoL) channel;

 - through an existing OpenBMC login session, and

 - via a physical UART connected to the BMC.

Regardless of the external interface used, the console data should be identical
between interface types.

Access to these sessions is not mutually exclusive. It's possible to have
multiple channels open at any one time, including multiple channels of the same
type. When multiple channels are open, serial data output from the host is
copied to all channels, and input from any channel is sent to the host.

### Direct ssh channel

The direct ssh channel interface allows console access over an encrypted
ssh session; connecting to the ssh session will immediately forward data
between the network channel and the host UART.

Console access is provided at a base port of 2200. If multiple consoles
are available, they will be exposed on contiguous port numbers starting
from 2200.

The specifics of the ssh protocol (authentication process, ciphers available,
etc.) will be the same as general ssh access to the OpenBMC login shell.

Authentication credentials for ssh access to the console may be separate from
those used for ssh access to the login shell. We recommend that a 'console'
role be implemented that grants access to the console.

### IPMI Serial over LAN

A console session may be accessed over an IPMI SoL session, if IPMI is supported
by the BMC.

### Existing login session

From an existing OpenBMC login session, the utility:

    obmc-console-client

Will open a console channel to the host.

### Physical UART

The console can also be made available to a hardware UART device, typically a
RS232 port.

Internal Interfaces
-------------------

The console system provides a few interfaces to OpenBMC infrastructure:

### Console data channel

The console multiplexer listens on a UNIX-domain socket, at address
`obmc-console` in the abstract `AF_UNIX` address space. This is a simple raw
console channel; `write()`s to this socket will be sent to the host, and data
from the host will be available via `read()`.

### Logging

A small backlog of serial data received from the host is kept in a limited-size
file, `/var/log/obmc-console.log`. By default, this file is capped at 16kB of
log data. Once this limit is reached, data will be discarded oldest-first.

### D-Bus

No D-Bus interfaces are exposed by the console infrastructure.

Implementation details
----------------------

The console connection to the host is via a UART-like channel. On current
platforms supported by OpenBMC, this is a 16550a-compatible serial device
exposed to the host over a LPC bus. As is typical with serial devices, the
16550a register set is mapped to base address 0x3f8.

The other "side" of this serial device is the BMC hardware. There may or may not
be a physical UART connection involved; ast2xxx hardware has a provision for a
"virtual" UART, which is essentially two sets of 16550a registers with their TX
& RX FIFOs connected directly. This has the advantage of not requiring baud rate
synchronisation, so we recommend this design is used on hardware that supports
it.

The BMC side of this serial connection is exposed as a normal Linux tty device
by the OpenBMC kernel (typically as `/dev/tty<NAME>`).

Data from this tty device needs to be multiplexed to client connections (to
support one of the access methods above). The `obmc-console` infrastructure
performs this multiplexing, where a single server interacts with the /dev/tty
connection from the host, and forwards data to (potentially multiple) client
connections.

Clients connect to the server over a UNIX-domain socket (at address
`obmc-console` in the abstract `AF_UNIX` namespace). Connections over this
socket simply pass raw console data between client and server. These clients
then allow external interfaces to send and receive console data from the
multiplexer.

Physical UARTs are supported directly in the server process, by copying data
between the host serial connection and a separate /dev/ttyXXX UART device.


                 [ /dev/ttyXXX ] : host serial connection
                        ^
                        |
                        v
              +-------------------+
              |obmc-console-server| ----> [ /dev/ttyXXX ] : physical UART
              +-------------------+
               ^        ^        ^
               |        |        |
               v        v        v
       +--------+   +--------+  +--------+
       | client |   | client |  | client |
       +--------+   +--------+  +--------+
           ^            ^            ^
           |            |            |
           v            v            v
       +--------+   +--------+  +--------+
       |  ssh   |   |  IPMI  |  |   ssh  |
       +--------+   +--------+  +--------+

### Console handlers

Internally, the obmc-console-server process uses separate handlers to process
data coming from the host. Currently, there are three handlers implemented:

 - socket-handler: exposes console data on the "obmc-console" UNIX-domain
   socket, to allow external processes (the SSH server and IPMI SoL server)
   to send and receive console data.

 - log-handler: writes incoming console data to the log file

 - tty-handler: transfers console data to another tty device, typically a
   separate hardware UART.

Additional handlers for extending console functionality within the server
process can be added fairly easily, by creating a `struct handler`, and
registering it (at link-time) with `console_register_handler()`.

For functionality that can exist in a separate process, use the
`obmc-console-client` utility; this simply forwards data between the socket
handler and stdin/stdout.

### Buffering multiplexed data

Because multiple clients can connect at once, the console infrastructure must
handle connections that may drain data at different rates.

To support this, we use a buffer to store data coming from the host, and drain
this buffer to each client in non-blocking mode. This allows bursts of console
data to be sent at a rate suitable for each handler.

[While we use a single buffer, each handler instance maintains its own pointers
into this buffer; this means that data can be consumed independently, without
requiring multiple copies of the data itself]

If the host is continuously sending data at a rate faster than the slowest
handler can drain, then the buffer may eventually fill. In this condition, the
console server will revert to blocking mode, and only read data from the host
when this slowest handler is able to drain it. The result of this is that the
multiplexer will not drop data; but the disadvantage is that continuous data
rates will be limited by the slowest handler.

In practicality, the slowest handler will generally be the physical UART, which
has a fixed baud rate. Therefore, we suggest that if a physical UART is used, it
be set to a high baud rate. 115200 is a good choice.

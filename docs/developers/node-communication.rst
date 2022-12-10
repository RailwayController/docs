.. SPDX-FileCopyrightText: 2022 Sidings Media <contact@sidingsmedia.com>
.. SPDX-License-Identifier: CC-BY-SA-4.0

Node Communication
==================

Communication between nodes is vital to ensure proper operation of the
control system. Due to the distributed nature of the system, a
standardised system to communicate between nodes is essential. There are
two ways in which nodes communicate within the network. Clients will
usually communicate with the main control board via the REST API
provided by the client bridge. Nodes will usually communicate with each
other over serial interfaces such as sockets, I2C and UART. In order to
ensure that communication is as smooth as posible, simple standards for
both serial communication and communication with the REST API have been
developed. The API is documented using the OpenAPI 3.1 specification and
the serial communication is defined using Augmented Backus–Naur form
(ABNF) as defined by `RFC 5234`_.

Registers
---------

Registers are conceptually similar to pigeonholes. In short, they are
named locations on each node that store specific pieces of configuration
data. These registers can be accessed and modified over the supported
communications protocols. These form the basis for each nodes API and
the registers are used to control the functionality of each node.

Each register has a locally unique address in the following format:

.. code-block:: abnf

    register-addr   =       1*(ALPHA / %x2D / %x5F)
                                ; Only support alphanumeric characters as
                                ; well as - and _ 

This address is used by commands to retrieve and modify the data in a
specific register.

.. note::
    Register addresses are case insensitive. I.e. speed_channel_1 is
    the same as SPEED_CHANNEL_1.

In cases where a request is made for the contents of a register but the
register is empty, ``null`` should be returned.


Reserved registers
^^^^^^^^^^^^^^^^^^

A number of addresses are reserved for use and MUST be present on all
nodes. They are used by the control nodes to establish the specific
features that an individual node supports and are essential to the
correct interoperation of all nodes.

``registers``
"""""""""""""

A comma seperated list of all supported register addresses available on
this node.

``serial``
""""""""""

A 16 character long string representing the serial number of the node.
The serial number is an arbitary string that MAY be unique among boards.
It is used solely for informational purposes. If no serial number is
defined, ``null`` SHOULD be returned.

``model``
"""""""""

The model number of this node. The model number is an arbitary string of
maximum length 256 characters that does not need to be unique. It is
used solely for informational purposes. If no model number is defined,
``null`` SHOULD be returned.

``bootloader``
""""""""""""""

A string representing the current bootloader version installed on this
node. This SHOULD be filled on all nodes. It is used to establish
compatibility of firmware and supported features.

``firmware``
""""""""""""

A string representing the current firmware version installed on the
node. This SHOULD be filled in on all nodes.

REST API
--------

This is predominantly used by clients to communicate with the main
control board over the users local network. A rendered version of the
OpenAPI specification is available from
https://docs.railwaycontroller.sidingsmedia.com/en/latest/_static/api.html.

Serial Commands
---------------

Serial commands are used for inter-node communication in almost all
cases. Most nodes are connected via serial communication mediums such as
I2C, UART and sockets. In these cases, the below specification for
serial commands should be used. 

These commands are loosely inspired by SQL statements. It is possible to
send multiple commands at once, seperated by the ``;`` character with a
carriage return and line feed being sent to indicate the end of all
commands. There are two types of command, the ``get`` command and the
``set`` command. As the names suggest, ``get`` commands retrieve a value
from a register and ``set`` commands set the value of a register.

In most cases, it is required to state the address of the node the
command is being sent to. This is to facilitate the command traversing
client bridges and interface cards. The only circumstance where the
address can be omitted is on commands sent by the main control board to
devices directly connected on the I2C bus. This is possible as the
address is already specified by the main control board when sending the
command over the I2C bus.

.. code-block:: abnf
    :caption: ABNF specification for serial command

    ; Commands
    command         =       1*query CRLF
                                ; Multiple commands may be sent at once.
                                ;   CRLF indicates end of commands

    query           =       (set / get) [SP addr] %x3B
                                ; SQL like format. Split queries using ;
                                ;   Address is only required when sending
                                ;   commands via an interface card. I.e.
                                ;   when being sent over the network. It is
                                ;   not required for direct I2C interfaces.
                                ;   Also used for commands between client 
                                ;   interface cards and the main controller

    get             =       "get" SP register-addr
                                ; GET commands used to retrieve data from
                                ;   registers

    set             =       "set" SP register-addr %x3D register-val
                                ; SET commands used to set the value of a
                                ;   register

    addr            =       "at" SP node-addr

    ; Command option values
    register-addr   =       string-val

    node-val        =       hex-val
                            / IPv6address

    register-val    =       string-val
                            / bin-val
                            / bool-val
                            / hex-val
                            / int-val
                            / signed-int-val   
                            / null-val

    string-val      =       1*(ALPHA / %x2D / %x5F)
                                ; Only support alphanumeric characters as
                                ; well as - and _ 

    bin-val         =       "0b" 1*BIT

    bool-val        =       "true" / "false"

    hex-val         =       "0x" 1*HEXDIG

    int-val         =       1*DIGIT

    signed-int-val  =       [%x2d] int

    null-val        =       "null"

    ;IPv6 Address from RFC5954
    IPv6address     =       6( h16 ":" ) ls32
                            / "::" 5( h16 ":" ) ls32
                            / [               h16 ] "::" 4( h16 ":" ) ls32
                            / [ *1( h16 ":" ) h16 ] "::" 3( h16 ":" ) ls32
                            / [ *2( h16 ":" ) h16 ] "::" 2( h16 ":" ) ls32
                            / [ *3( h16 ":" ) h16 ] "::"    h16 ":"   ls32
                            / [ *4( h16 ":" ) h16 ] "::"              ls32
                            / [ *5( h16 ":" ) h16 ] "::"              h16
                            / [ *6( h16 ":" ) h16 ] "::"

    h16             =       1*4HEXDIG

    ls32            =       ( h16 ":" h16 ) / IPv4address

    IPv4address     =       dec-octet "." dec-octet "." dec-octet "." dec-octet

    dec-octet       =       DIGIT                   ; 0-9
                            / %x31-39 DIGIT         ; 10-99
                            / "1" 2DIGIT            ; 100-199
                            / "2" %x30-34 DIGIT     ; 200-249
                            / "25" %x30-35          ; 250-255


.. _`RFC 5234`: https://www.rfc-editor.org/rfc/rfc5234.html

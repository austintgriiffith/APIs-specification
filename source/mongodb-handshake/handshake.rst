=================
MongoDB Handshake
=================

:Spec Title: MongoDB Handshake
:Spec Version: 1.2.0
:Author: Hannes Magnusson
:Kernel Advisory: Mark Benvenuto
:Driver Advisory: Anna Herlihy, Justin Lee
:Status: Approved
:Type: Standards
:Minimum Server Version: 3.4
:Last Modified: 2020-02-12


.. contents::

--------


Abstract
========

MongoDB 3.4 has the ability to annotate connections with metadata provided by
the connecting client. The intent of this metadata is to be able to identify
client level information about the connection, such as application name, driver
name and version. The provided information will be logged through the
mongo[d|s].log and the profile logs; this should enable sysadmins to easily
backtrack log entries the offending application. The active connection data
will also be queryable through aggregation pipeline, to enable collecting and
analyzing driver trends.

After connecting to a MongoDB node an isMaster command is issued, followed by
authentication, if appropriate. This specification augments this handshake and
defines certain arguments that clients provide as part of the handshake.

This spec furthermore adds a new connection string argument for applications to
declare its application name to the server.

META
====

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in `RFC 2119 <https://www.ietf.org/rfc/rfc2119.txt>`_.


Specification
=============

--------------------
Connection handshake
--------------------

The ``isMaster`` handshake MUST be performed on every socket to any and all servers
upon establishing the connection to MongoDB, including reconnects of dropped
connections and newly discovered members of a cluster. It MUST be the first
command sent over the respective socket. If the command fails the client MUST
disconnect.

``isMaster`` commands issued after the initial connection handshake MUST NOT
contain handshake arguments. Any subsequent ``isMaster`` calls, such as the ones
for topology monitoring purposes, MUST NOT include this argument.


The ``isMaster`` handshake, as of MongoDB 3.4, supports a new argument, ``client``,
provided as a BSON object. This object has the following structure::

    {
        isMaster: 1,
        client: {
            /* OPTIONAL. If present, the "name" is REQUIRED */
            application: {
                name: "<string>"
            },
            /* REQUIRED, including all sub fields */
            driver: {
                name: "<string>",
                version: "<string>"
            },
            /* REQUIRED */
            os: {
                type: "<string>",         /* REQUIRED */
                name: "<string>",         /* OPTIONAL */
                architecture: "<string>", /* OPTIONAL */
                version: "<string>"       /* OPTIONAL */
            },
            /* OPTIONAL */
            platform: "<string>"
        }
    }




client.application.name
~~~~~~~~~~~~~~~~~~~~~~~

This value is application configurable.

The application name is printed to the mongod logs upon establishing the
connection. It is also recorded in the slow query logs and profile collections.

The recommended way for applications to provide this value is through the
connection URI. The connection string key is ``appname``.

Example connection string::

   mongodb://server:27017/db?appname=mongodump

This option MAY also be provided on the MongoClient itself, if normal for the
driver. It is only valid to set this attribute before any connection has been
made to a server. Any attempt to set ``client.application.name`` MUST result in an
failure when doing so will either change the existing value, or have any
connection to MongoDB reporting inconsistent values.

Drivers MUST NOT provide a default value for this key.


client.driver.name
~~~~~~~~~~~~~~~~~~

This value is required and is not application configurable.

The internal driver name. For drivers written on-top of other core drivers, the
underlying driver will typically expose a function to append additional name to
this field.

Example::

        - "pymongo"
        - "mongoc / phongo"


client.driver.version
~~~~~~~~~~~~~~~~~~~~~

This value is required and is not application configurable.

The internal driver version. The version formatting is not defined. For drivers
written on-top of other core drivers, the underlying driver will typically
expose a function to append additional name to this field.

Example::

        - "1.1.2-beta0"
        - "1.4.1 / 1.2.0"


client.os.type
~~~~~~~~~~~~~~

This value is required and is not application configurable.

The Operating System primary identification type the client is running on.
Equivalent to ``uname -s`` on POSIX systems.  This field is REQUIRED and clients
must default to ``unknown`` when an appropriate value cannot be determined.

Example::

        - "Linux"
        - "Darwin"
        - "Windows"
        - "BSD"
        - "Unix"


client.os.name
~~~~~~~~~~~~~~

This value is optional, but RECOMMENDED, it is not application configurable.

Detailed name of the Operating System???s, such as fully qualified distribution
name. On systemd systems, this is typically ``PRETTY_NAME`` of ``os-release(5)``
(``/etc/os-release``) or the ``DISTRIB_DESCRIPTION`` (``/etc/lsb-release``,
``lsb_release(1) --description``) on LSB systems. The exact value and method to
determine this value is undefined.

Example::

        - "Ubuntu 16.04 LTS"
        - "macOS"
        - "CygWin"
        - "FreeBSD"
        - "AIX"


client.os.architecture
~~~~~~~~~~~~~~~~~~~~~~

This value is optional, but RECOMMENDED, it is not application configurable.
The machine hardware name. Equivalent to ``uname -m`` on POSIX systems.

Example::

        - "x86_64"
        - "ppc64le"


client.os.version
~~~~~~~~~~~~~~~~~

This value is optional and is not application configurable.

The Operating System version.

Example::

        - "10"
        - "8.1"
        - "16.04.1"


client.platform
~~~~~~~~~~~~~~~

This value is optional and is not application configurable.

Driver specific platform details.

Example::

        - clang 3.8.0 CFLAGS="-mcpu=power8 -mtune=power8 -mcmodel=medium"
        - "Oracle JVM EE 9.1.1"


--------------------------
Speculative Authentication
--------------------------

:since: 4.4

The ``isMaster`` handshake supports a new argument, ``speculativeAuthenticate``,
provided as a BSON document. Clients specifying this argument to ``isMaster`` will
speculatively include the first command of an authentication handshake.
This command may be provided to the server in parallel with any standard request for
supported authentication mechanisms (i.e. ``saslSupportedMechs``). This would permit
clients to merge the contents of their first authentication command with their
``isMaster`` request, and receive the first authentication reply along with the
``isMaster`` reply.

When the mechanism is ``MONGODB-X509``, ``speculativeAuthenticate`` has the same
structure as seen in the MONGODB-X509 conversation section in the
`Driver Authentication spec <https://github.com/mongodb/specifications/blob/master/source/auth/auth.rst#supported-authentication-methods>`_.

When the mechanism is ``SCRAM-SHA-1`` or ``SCRAM-SHA-256``, ``speculativeAuthenticate``
has the same fields as seen in the conversation subsection of the SCRAM-SHA-1 and
SCRAM-SHA-256 sections in the `Driver Authentication spec <https://github.com/mongodb/specifications/blob/master/source/auth/auth.rst#supported-authentication-methods>`_
with an additional ``db`` field to specify the name of the authentication database.

If the ``isMaster`` command with a ``speculativeAuthenticate`` argument succeeds,
the client should proceed with the next step of the exchange. If the ``isMaster``
response does not include a ``speculativeAuthenticate`` reply and the ``ok`` field
in the ``isMaster`` response is set to 1, drivers MUST authenticate using the standard
authentication handshake.

The ``speculativeAuthenticate`` reply has the same fields, except for the ``ok`` field,
as seen in the conversation sections for MONGODB-X509, SCRAM-SHA-1 and SCRAM-SHA-256
in the `Driver Authentication spec <https://github.com/mongodb/specifications/blob/master/source/auth/auth.rst#supported-authentication-methods>`_.

If an authentication mechanism is not provided either via connection string or code, but
a credential is provided, drivers MUST use the SCRAM-SHA-256 mechanism for speculative
authentication and drivers MUST send ``saslSupportedMechs``.

Older servers will ignore the ``speculativeAuthenticate`` argument. New servers will
participate in the standard authentication conversation if this argument is missing.


Supporting Wrapping Libraries
=============================

Drivers MUST allow libraries which wrap the driver to append to the client
metadata generated by the driver. The following class definition defines the
options which MUST be supported:

.. code:: typescript

    class DriverInfoOptions {
        /**
        * The name of the library wrapping the driver.
        */
        name: String;

        /**
        * The version of the library wrapping the driver.
        */
        version: Optional<String>;

        /**
        * Optional platform information for the wrapping driver.
        */
        platform: Optional<String>;
    }


Note that how these options are provided to a driver is left up to the implementor.

If provided, these options MUST NOT replace the values used for metadata generation.
The provided options MUST be appended to their respective fields, and be delimited by
a ``|`` character. For example, when `Motor <https://docs.mongodb.com/ecosystem/drivers/motor/>`_
wraps PyMongo, the following fields are updated to include Motor's "driver info":

.. code:: typescript

    {
        client: {
            driver: {
                name: "PyMongo|Motor",
                version: "3.6.0|2.0.0"
            }
        }
    }


**NOTE:** All strings provided as part of the driver info MUST NOT contain the delimiter used
for metadata concatention. Drivers MUST throw an error if any of these strings contains that
character.

----------
Deviations
----------

Some drivers have already implemented such functionality, and should not be required to make
breaking changes to comply with the requirements set forth here. A non-exhaustive list of
acceptable deviations are as follows:

* The name of `DriverInfoOptions` is non-normative, implementors may feel free to name this whatever they like.
* The choice of delimiter is not fixed, ``|`` is the recommended value, but some drivers currently use ``/``.
* For cases where we own a particular stack of drivers (more than two), it may be preferable to accept a *list* of strings for each field.

Limitations
===========

The entire metadata BSON document MUST NOT exceed 512 bytes. This includes all
BSON overhead.  The ``client.application.name`` cannot exceed 128 bytes.  MongoDB
will return an error if these limits are not adhered to, which will result in
handshake failure. Drivers MUST validate these values and truncate driver
provided values if necessary. Implementors are encouraged to prioritize truncating
the ``platform`` field before all others. Additionally, implementors are
encouraged to place high priority information about the platform earlier in the
string, in order to avoid possible truncating of those details.

Test Plan
=========

Unknown. A set of YAML tests for the connection uri. It???ll implicitly test the
other fields being provided.

Motivation For Change
=====================

Being able to annotate individual connections with custom data will allow users
and sysadmins to easily correlate events happening on their MongoDB deployment
to a specific application. For support engineers, it furthermore helps identify
potential problems in drivers or client platforms, and paves the way for
providing proactive support via Cloud Manager and/or Atlas to advise customers
about out of date driver versions.


Design Rationale
================

Drivers run on a multitude of platforms, languages, environments and systems.
There is no defined list of data points that may or may not be valuable to
every system. Rather than specifying such a list it was decided we would report
the basics; something that everyone can discover and consider valuable. The
obvious requirement here being the driver itself and its version. Any
additional information is generally very system specific. Scala may care to
know the Java runtime, while Python would like to know if it was built with C
extensions - and C would like to know the compiler.

Having to define dozens of arguments that may or may not be useful to one or
two drivers isn???t a good idea. Instead, we define a ``platform`` argument that is
driver dependent. This value will not have defined value across drivers and is
therefore not generically queryable -- however, it will gain defined schema for
that particular driver, and will therefore over time gain defined structure
that can be formatted and value extracted from.

Backwards Compatibility
=======================

The ``isMaster`` command currently ignores arguments (i.e. If arguments are
provided the ``isMaster`` command discards them without erroring out). This
functionality has therefore no backwards compatibility concerns.

Reference Implementation
========================

`C Driver <https://github.com/mongodb/mongo-c-driver/blob/master/src/mongoc/mongoc-handshake.c>`_.

Q&A
===

* The 128 bytes application.name limit, does that include BSON overhead
   * No, just the string itself
* The 512 bytes limit, does that include BSON overhead?
   * Yes
* The 512 bytes limit, does it apply to the full ``isMaster`` document or just the ``client`` subdocument
   * Just the subdocument
* Should I really try to fill the 512 bytes with data?
   * Not really. The server does not attempt to normalize or compress this data in anyway, so it will hold it in memory as-is per connection. 512 bytes for 20,000 connections is ~ 10mb of memory the server will need.
* What happens if I pass this new ``isMaster`` argument to previous MongoDB versions?
   * Nothing. Arguments passed to ``isMaster`` prior to MongoDB 3.4 are not treated in any special way and have no effect one way or other
* Are there wire version bumps or anything accompanying this specification?
   * No
* Is establishing the handshake required for connecting to MongoDB 3.4?
   * No, it only augments the connection. MongoDB will not reject connections without it
* Does this affect SDAM implementations?
   * Possibly. There are a couple of gotchas. If the application.name is not in the URI...
      * The SDAM monitoring cannot be launched until the user has had the ability
        to set the application name because the application name has to be sent to the
        first ``isMaster``. This means that the connection pool cannot be established until
        the first user initiated command, or else some connections will have the
        application name while other won???t
      * The ``isMaster`` handshake must be called on all sockets, including
        administrative background sockets to MongoDB
* My language doesn't have ``uname``, but does instead provide its own variation of these values, is that OK?
   * Absolutely. As long as the value is identifiable it is fine. The exact method and values are undefined by this specification

Changes
=======

* 2019-11-13: Added section about supporting wrapping libraries
* 2020-02-12: Added section about speculative authentication

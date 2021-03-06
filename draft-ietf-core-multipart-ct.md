---
stand_alone: true
ipr: trust200902
docname: draft-ietf-core-multipart-ct-latest
cat: std
pi:
  strict: 'yes'
  toc: 'yes'
  tocdepth: '4'
  symrefs: 'yes'
  sortrefs: 'yes'
  compact: 'yes'
  subcompact: 'no'
  comments: 'yes'
  inline: 'yes'
title: Multipart Content-Format for CoAP
abbrev: Multipart Content-Format for CoAP
area: ART
wg: CoRE
kw: CoAP, Multipart Content-Format
date: 2018-08-07
author:
- ins: T. F. Fossati
  name: Thomas Fossati
  org: Nokia
  email: thomas.fossati@nokia.com
-
  name: Klaus Hartke
  org: Ericsson
  email: klaus.hartke@ericsson.com
  street: Torshamnsgatan 23
  city: Stockholm
  code: SE-16483
  country: Sweden
-
  ins: C. Bormann
  name: Carsten Bormann
  org: Universität Bremen TZI
  street: Postfach 330440
  city: Bremen
  code: D-28359
  country: Germany
  phone: +49-421-218-63921
  email: cabo@tzi.org

normative:
  RFC7049: cbor
  RFC7252: coap
  RFC7641: observe
informative:
  I-D.ietf-ace-coap-est: est-coap
  I-D.ietf-cbor-cddl: cddl

--- abstract

This memo defines application/multipart-core, an
application-independent media-type that can be used
to combine representations of several different media types into a single
CoAP message-body with minimal framing overhead, each along with a CoAP
Content-Format identifier.

--- middle

# Introduction

This memo defines application/multipart-core, an application-independent
media-type that can be used to combine representations of several different media types
into a single CoAP {{-coap}} message-body with minimal framing
overhead, each along with a CoAP Content-Format identifier.

This simple and efficient binary framing mechanism can be employed to
create application specific request and response bodies which build on
multiple already existing media types.

Applications using the application/multipart-core Content-Format define the
internal structure of the application/multipart-core representation.

For example, one way to structure the sub-types specific to an application/multipart-core
container is to always include them at the same fixed position.
This specification allows to indicate that an optional part is not
present by substituting a null value for the representation of the part.

Optionally, an application might use the general format defined here,
but also register a new media type and an associated Content-Format
identifier --- typically one in the range 10000-64999 --- instead of
using application/multipart-core.


<!--  Leave out until needed:

## Requirements Language

{: :boilerplate bcp14}
-->

# Multipart Content-Format Encoding

A representation of media-type application/multipart-core contains a collection of
zero or more representations, each along with their respective content
format.

The collection is encoded as a CBOR {{-cbor}} array with an even
number of elements.
The second, fourth, sixth, etc. element is a byte string containing
a representation, or the value `null` if an optional part is indicated
as not given.
The first, third, fifth, etc. element is an unsigned integer
specifying the content format ID of the representation following it.
(Future extensions might want to include additional alternative ways of
specifying the media type of a representation in such a position.)

For example, a collection containing two representations, one with
content format ID 42 and one with content format ID 0, looks like this
in CBOR diagnostic notation:

> \[42, h'0123456789abcdef', 0, h'3031323334']

For illustration, the structure of an application/multipart-core representation can
be described by the CDDL {{-cddl}} specification in {{mct-cddl}}:

~~~CDDL
multipart-core = [* multipart-part]
multipart-part = (type: uint .size 2, part: bytes / null)
~~~
{:#mct-cddl title="CDDL for application/multipart-core"}

# Usage Examples

## Observing Resources

When a client registers to observe a resource {{-observe}} for which no
representation is available yet, the server may send one or more 2.05
(Content) notifications before sending the first actual 2.05
(Content) or 2.03 (Valid) notification.  The possible resulting
sequence of notifications is shown in Figure 1.

          __________       __________       __________
         |          |     |          |     |          |
    ---->|   2.05   |---->|  2.05 /  |---->|  4.xx /  |
         | Pending  |     |   2.03   |     |   5.xx   |
         |__________|     |__________|     |__________|
            ^   \ \          ^    \           ^
             \__/  \          \___/          /
                    \_______________________/
{: #fig-sequence title="Sequence of Notifications:"}

The specification of the Observe option requires that all
notifications carry the same Content-Format.  The
application/multipart-core media type can be used to provide that
Content-Format: e.g., carrying an empty list of representations in the
case marked as "Pending" in {{fig-sequence}}, and carrying a single
representation specified as the target content-format in the case in
the middle of the figure.

Implementation hints
--------------------

This section describes the serialization for readers that may be new
to CBOR.  It does not contain any new information.

An application/multipart-core representation carrying no
representations is represented by an empty CBOR array, which is
serialized as a single byte with the value 0x80.

An application/multipart-core representation carrying a single
representation is represented by a two-element CBOR array, which is
serialized as 0x82 followed by the two elements.  The first element is
an unsigned integer for the Content-Format value, which is represented as described in
{{tbl-integer}}.  The second element is the object as a byte string,
which is represented as a length as described in {{tbl-length}}
followed by the bytes of the object.

| Serialization  |      Value |
| 0x00..0x17     |      0..23 |
| 0x18 0xnn      |    24..255 |
| 0x19 0xnn 0xnn | 256..66535 |
{: #tbl-integer title="Serialization of content-format"}

| Serialization  |     Length |
| 0x40..0x57     |      0..23 |
| 0x58 0xnn      |    24..255 |
| 0x59 0xnn 0xnn | 256..66535 |
| 0x5a 0xnn 0xnn 0xnn 0xnn | 66536..4294967295  |
| 0x5b 0xnn .. 0xnn (8 bytes)  | 4294967296.. |
{: #tbl-length title="Serialization of object length"}

For example, a single text/plain object (content-format 0) of value
"Hello World" (11 characters) would be serialized as

> 0x82 0x00 0x4b H e l l o 0x20 w o r l d

In effect, the serialization for a single object is done by prefixing
the object with information about its content-format (here: 0x82 0x00)
and its length (here: 0x4b).

For more than one representation included in an
application/multipart-core representation, the head of the CBOR array
is adjusted (0x84 for two representations, 0x86 for three, ...) and
the sequences of content-format and embedded representations follow.

# IANA Considerations {#IANA}

## Registration of media type application/multipart-core

IANA is requested to register the following media type {{?RFC6838}}:

Type name:
: application

Subtype name:
: multipart-core

Required parameters:
: N/A

Optional parameters:
: N/A

Encoding considerations:
: binary

Security considerations:
: See the Security Considerations Section of RFCthis

Interoperability considerations:
: N/A

Published specification:
: RFCthis

Applications that use this media type:
: Applications that need to combine representations of potentially
  several media types into one, e.g., EST-CoAP {{-est-coap}}

Fragment identifier considerations:
: N/A

Additional information:
:     Deprecated alias names for this type:
      : N/A

      Magic number(s):
      : N/A

      File extension(s):
      : N/A

      Macintosh file type code(s):
      : N/A

Person & email address to contact for further information:
: iesg&ietf.org

Intended usage:
: COMMON

Restrictions on usage:
: N/A

Author:
: CoRE WG

Change controller:
: IESG

Provisional registration? (standards tree only):
: no


## Registration of a Content-Format identifier for application/multipart-core

IANA is requested to register the following Content-Format to the
"CoAP Content-Formats" subregistry, within the "Constrained RESTful
Environments (CoRE) Parameters" registry, from the Expert Review space
(0..255):

   | Media Type                 | Encoding | ID   | Reference |
   | application/multipart-core | ---      | TBD1 | RFCthis   |

# Security Considerations {#Security}

The security considerations of {{-cbor}} apply.  In particular,
resource exhaustion attacks may employ large values for the byte string
size fields, or deeply nested structures of recursively embedded
application/multipart-core representations.

--- back


# Acknowledgements
{: numbered='no'}

Most of the text in this draft is from earlier contributions by two of
the authors, Thomas Fossati and Klaus Hartke.  The re-mix in this
document is based on the requirements in {{-est-coap}}, based on
discussions with Michael Richardson, Panos Kampanis and Peter van der
Stok.


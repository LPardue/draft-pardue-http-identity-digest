---
title: "HTTP Identity Digest"
abbrev: "HTTP Identity Digest"
category: std

docname: draft-pardue-http-identity-digest-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Applications"
workgroup: "HyperText Transfer Protocol"
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: "HyperText Transfer Protocol"
  type: "Working Group"
  mail: "http-wg@hplb.hp.com"
  arch: "https://www.ics.uci.edu/pub/ietf/http/hypermail"
  github: "LPardue/draft-pardue-http-identity-digest"
  latest: "https://LPardue.github.io/draft-pardue-http-identity-digest/draft-pardue-http-identity-digest.html"

author:
 -
    fullname: Lucas Pardue
    organization: Cloudflare
    email: lucaspardue.24.7@gmail.com
 -
    fullname: Mike West
    organization: Google
    email: mkwst@google.com

normative:

informative:


--- abstract

The Repr-Digest and Content-Digest integrity fields are subject to HTTP content
coding considerations. There are some use cases that benefit from the
unambiguous exchange of integrity digests of unencoded representation. The
Identity-Digest and Want-Identity-Digest fields complement existing integrity
fields for this purpose.


--- middle

# Introduction

The Integrity fields defined in {{!DIGEST-FIELDS=RFC9530}} are suitable for a
range of use cases. However, because the fields are subject to HTTP content
coding considerations, it is difficult to support use cases that could benefit
from the exchange of integrity digests of the unencoded representation.

As a simple example, an application using HTTP might be presented with request
or response representation data that has been transparently decoded.  Attempting
to verify the integrity of the data against the Repr-Digest would first require
re-encoding that data using the same coding indicated by the Content-Encoding
header field ({{Section 8.4 of !HTTP=RFC9110}}), which is not always possible
(see {{Section 6.5 of DIGEST-FIELDS}}).

Even when receiver-side re-encoding for the purpose of Repr-Digest validation is
technically possible, it might not be practical for certain kinds of
environments. For instance, browsers tend to provide built-in support for
transparent decoding but little support for encoding; while this could be done
via the use of additional libraries it would create work in JavaScript that
could contend with other activities. Even on the server side, the re-encoding of
received data might not be acceptable; some coding algorithms are optimized
towards efficient decoding at the cost of complex encoding. This is all made
more complex if the the Content-Encoding field value indicates a series of
encodings.

A more complex example involves HTTP Range Requests ({{Section 14 of
HTTP}}), where a client fetches multiple partial representations from
different origins and "stitches" them back into a whole. Unfortunately, if the
origins apply different content coding, the Repr-Digest field will vary by the
server's selected encoding (i.e. the Content-Encoding header field, {{Section
8.4 of HTTP}}). This provides a challenge for a client - in order to verify the
integrity of the pieced-together whole it would need to remove the encoding of
each part, combine them, and then encode the result in order to compare against
one or more Repr-Digests.

The Accept-Encoding header field ({{Section 12.5.3 of HTTP}}) provides the means
to indicate preferences for content coding. It is possible for an endpoint to
indicate a preference for no encoding, for example by sending the "identity"
token. However, codings often provide data compression that is advantageous.
Disabling content coding in order to simplify integrity checking is possibly an
unacceptable trade off.

For a variety of reasons, decoding and re-encoding content in order to benefit
from HTTP integrity fields is not preferable. This specification defines the
Identity-Digest and Want-Identity-Digest fields to support a simpler validation
workflow in some scenarios where content coding is applied. These fields
complement the other integrity fields defined in {{DIGEST-FIELDS}}.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the Augmented BNF defined in {{!RFC5234}} and updated by
{{!RFC7405}}. This includes the rules: LF (line feed)

This document uses the following terminology from {{Section 3 of
!STRUCTURED-FIELDS=RFC9651}} to specify syntax and parsing: Byte Sequence,
Dictionary, Integer, and List.

The definitions "representation", "selected representation", "representation
data", "representation metadata", "user agent" and "content" in this document
are to be interpreted as described in {{!HTTP=RFC9110}}.

Integrity fields: collective term for `Content-Digest`, `Repr-Digest`, and `Identity-Digest`

Integrity preference fields: collective term for `Want-Repr-Digest`,  `Want-Content-Digest`, and `Want-Identity-Digest`

# Complementary Integrity Fields

The following examples illustrate how Integrity fields can be used in
combination to address different and complementary needs, particularly the cases
described in {{introduction}}. The unencoded data used in the example is the
string "An unexceptional string" following by an LF character.

When a response message is not conveying partial or encoded representation data,
all Integrity fields contain the same value, making validation trivial and
identical.

~~~ http-message
GET /boringstring HTTP/1.1
Host: example.org

~~~
{: title="Simple GET request"}

~~~ http-message
NOTE: '\' line wrapping per RFC 8792

HTTP/1.1 200 OK
Content-Length: 24
Content-Digest: \
  sha-256=:5Bv3NIx05BPnh0jMph6v1RJ5Q7kl9LKMtQxmvc9+Z7Y=:
Repr-Digest: \
  sha-256=:5Bv3NIx05BPnh0jMph6v1RJ5Q7kl9LKMtQxmvc9+Z7Y=:
Identity-Digest: \
  sha-256=:5Bv3NIx05BPnh0jMph6v1RJ5Q7kl9LKMtQxmvc9+Z7Y=:

An unexceptional string
~~~
{: title="Response to GET request"}

When a response message conveys complete encoded content, the Content-Digest and
the Repr-Digest are the same, while the Identity-Digest is different.

~~~ http-message
GET /boringstring HTTP/1.1
Host: example.org
Accept-Encoding: gzip

~~~
{: title="GET request with content negotiation"}

~~~ http-message
NOTE: '\' line wrapping per RFC 8792

HTTP/1.1 200 OK
Content-Encoding: gzip
Content-Digest: \
  sha-256=:XyjvEuFb1P5rqc2le3vQm7M96DwZhvmOwqHLu2xVpY4=:
Repr-Digest: \
  sha-256=:XyjvEuFb1P5rqc2le3vQm7M96DwZhvmOwqHLu2xVpY4=:
Identity-Digest: \
  sha-256=:5Bv3NIx05BPnh0jMph6v1RJ5Q7kl9LKMtQxmvc9+Z7Y=:

1f 8b 08 00 79 1f 08 64 00 ff
73 cc 53 28 cd 4b ad 48 4e 2d
28 c9 cc cf 4b cc 51 28 2e 29
ca cc 4b e7 02 00 7e af 07 44
18 00 00 00
~~~
{: title="Response with gzip encoding"}

Finally, when a response message contains partial and encoded content, all
Integrity fields vary. The Content-Digest can be used to validate the integrity
of the received part. Repr-Digest or Identity-Digest can be used later after
reconstruction, the choice of which to use is left to the application, which
would consider a range of factors outside the scope of discussion.

~~~ http-message
GET /boringstring HTTP/1.1
Host: example.org
Accept-Encoding: gzip
Range: bytes=0-10

~~~
{: title="Range request with content negotiation"}

~~~ http-message
NOTE: '\' line wrapping per RFC 8792

HTTP/1.1 206 Partial Content
Content-Encoding: gzip
Content-Digest: \
  sha-256=:SotB7Pa5A7iHSBdh9mg1Ev/ktAzrxU4Z8ldcCIUyfI4=:
Repr-Digest: \
  sha-256=:XyjvEuFb1P5rqc2le3vQm7M96DwZhvmOwqHLu2xVpY4=:
Identity-Digest: \
  sha-256=:5Bv3NIx05BPnh0jMph6v1RJ5Q7kl9LKMtQxmvc9+Z7Y=:

1f 8b 08 00 79 1f 08 64 00 ff
~~~
{: title="Partial response with gzip encoding"}

# The Identity-Digest Field {#identity-digest}

The Identity-Digest HTTP field can be used in requests and responses to
communicate digests that are calculated using a hashing algorithm applied to the
representation with no content coding (a.k.a. an identity encoding). Apart
from the content coding concerns, it behaves similarly to Repr-Digest.

Identity-Digest is a `Dictionary` (see {{Section 3.2 of STRUCTURED-FIELDS}})
where each:

* key conveys the hashing algorithm (see {{Section 5 of DIGEST-FIELDS}} used to
  compute the digest;
* value is a `Byte Sequence` ({{Section 3.3.5 of STRUCTURED-FIELDS}}), that
  conveys an encoded version of the byte output produced by the digest
  calculation.

For example:

~~~ http-message
NOTE: '\' line wrapping per RFC 8792

Identity-Digest: \
  sha-512=:YMAam51Jz/jOATT6/zvHrLVgOYTGFy1d6GJiOHTohq4yP+pgk4vf2aCs\
  yRZOtw8MjkM7iw7yZ/WkppmM44T3qg==:
~~~

The `Dictionary` type can be used, for example, to attach multiple digests
calculated using different hashing algorithms in order to support a population
of endpoints with different or evolving capabilities. Such an approach could
support transitions away from weaker algorithms (see
{{Section 6.6 of DIGEST-FIELDS}}).

~~~ http-message
NOTE: '\' line wrapping per RFC 8792

Identity-Digest: \
  sha-256=:d435Qo+nKZ+gLcUHn7GQtQ72hiBVAgqoLsZnZPiTGPk=:,\
  sha-512=:YMAam51Jz/jOATT6/zvHrLVgOYTGFy1d6GJiOHTohq4yP+pgk4vf2aCs\
  yRZOtw8MjkM7iw7yZ/WkppmM44T3qg==:
~~~

A recipient MAY ignore any or all digests. Application-specific behavior or
local policy MAY set additional constraints on the processing and validation
practices of the conveyed digests. The security considerations cover some of
the issues related to ignoring digests (see {{Section 6.6 of DIGEST-FIELDS}})
and validating multiple digests (see {{Section 6.7 of DIGEST-FIELDS}}).

A sender MAY send a digest without knowing whether the recipient supports a
given hashing algorithm. A sender MAY send a digest if it knows the recipient
will ignore it.

Identity-Digest can be sent in a trailer section. In this case, Identity-Digest
MAY be merged into the header section; see {{Section 6.5.1 of HTTP}}.

# The Want-Identity-Digest Field

Want-Identity-Digest indicates that the sender would like to receive a
representation digest on messages associated with the request URI and
representation metadata where no content coding is applied, using the
Identity-Digest field.

If Want-Identity-Digest is used in a response, it indicates that the server
would like the client to provide the Identity-Digest field on future requests.

Want-Identity-Digest is only a hint. The receiver of the field can ignore it and
send an Integrity field using any algorithm or omit fields entirely. It is not a
protocol error if preferences are ignored. Applications that use Integrity
fields and Integrity preferences can define expectations or constraints that
operate in addition to this specification.

Want-Identity-Digest is of type `Dictionary` where each:

* key conveys the hashing algorithm;
* value is an `Integer` ({{Section 3.3.1 of STRUCTURED-FIELDS}}) that conveys an
  ascending, relative, weighted preference. It must be in the range 0 to 10
  inclusive. 1 is the least preferred, 10 is the most preferred, and a value of
  0 means "not acceptable".

Examples:

~~~ http-message
Want-Identity-Digest: sha-256=1
Want-Identity-Digest: sha-512=3, sha-256=10, unixsum=0
~~~

# Security Considerations

The considerations in {{DIGEST-FIELDS}} apply. There are no known additional
considerations.


# IANA Considerations

This document has no IANA actions (yet)


--- back

# Acknowledgments
{:numbered="false"}

Early drafts of {{DIGEST-FIELDS}} included a mechanism to support the exchange
of digests where no content coding is applied, which was removed before
publication. While the design here is different, it is motivated by discussion
of the previous design in the HTTP WG. The motivating use cases still mostly
apply identically.

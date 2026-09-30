---
title: 'The "at" URI Scheme'
abbrev: "AT URI"
category: std

docname: draft-newbold-atp-aturi-latest
submissiontype: IETF
number:
updates:
date:
consensus:
v: 0
area: "Applications and Real-Time Area"
workgroup: "Authenticated Transfer"
keyword:
venue:
  group: "Authenticated Transfer"
  type: "Working Group"
  mail: "atp@ietf.org"
  github: "ietf-wg-atp/drafts"

author:
 -
    fullname: Bryan Newbold
    organization: Bluesky Social
    email: bnewbold@robocracy.org

normative:
  RFC3986: RFC3986
  RFC5234: RFC5234
  RFC7595: RFC7595
  AT-REPOSYNC:
    title: "Authenticated Transfer: Repository and Synchronization"
    date: September 2026
    target: https://datatracker.ietf.org/doc/draft-holmgren-at-repository/
    author:
      -
        fullname: Daniel Holmgren
        organization: Bluesky Social
      -
        fullname: Bryan Newbold
        organization: Bluesky Social

informative:
  AT-ARCH:
    title: "Authenticated Transfer: Architecture Overview"
    date: March 2026
    target: https://datatracker.ietf.org/doc/draft-newbold-at-architecture
    author:
      -
        fullname: Bryan Newbold
        organization: Bluesky Social
      -
        fullname: Daniel Holmgren
        organization: Bluesky Social
  ATPAPER:
    title: "Bluesky and the AT Protocol: Usable Decentralized Social Media"
    date: December 2024
    target: https://doi.org/10.1145/3694809.3700740
    author:
      -
        fullname: Martin Kleppmann
      -
        fullname: Paul Frazee
      -
        fullname: Jake Gold
      -
        fullname: Jay Graber
      -
        fullname: Daniel Holmgren
      -
        fullname: Devin Ivy
      -
        fullname: Jeromy Johnson
      -
        fullname: Bryan Newbold
      -
        fullname: Jaz Volpert
...

--- abstract

This document defines the "at" URI scheme, which is used to reference accounts and data records in the Authenticated Transfer Protocol (ATP).

--- middle

# Introduction {#intro}

The Authenticated Transfer Protocol (ATP) enables the creation of decentralized networks for publication of self-certifying data. An introduction to the overall protocol architecture is given in {{AT-ARCH}}, and the data repository and synchronization mechanisms are described in {{AT-REPOSYNC}}.

Each account has a global permanent account identifier that can be resolved to a network hosting location and to public key material. Multiple account identifier systems are supported, but details are out of scope for this document. Accounts publish structured data records of different application-defined types in data repositories. Each account has a single repository for all of its public data records, organized in collections by record type, with one or more records in each collection.

Individual records can be globally referenced by account, collection, and record key. Records themselves may include references to other records, forming a global data graph. Record references can be resolved to fetch individual records. They can also be used to annotate records for the purpose of content moderation.

This document describes a string identifier syntax for data record references.

The identifiers described in this version of the document comply with most of the {{RFC3986}} generic Uniform Resource Identifier (URI) requirements, but not all of them. Using account identifiers with multiple colons in the authority section violates the generic syntax rules for URI schemes using the double-slash prefix (`//`), which this version uses. This means the identifier syntax described in this document is not eligible for inclusion in the IANA URI Registry under {{RFC7595}}.

# Structure {#structure}

The generic structure of an "at" URI is:

~~~
"at://" ACCOUNT-AUTHORITY [ PATH ] [ "?" QUERY ] [ "#" FRAGMENT ]
~~~

The required authority section references an account. It may be a permanent account identifier or an account handle. A URI that only includes the authority section can be used as a reference to an overall account. Handles in the authority section are discouraged in most other use cases; see {{security}}.

The structure aligns with the generic structure and semantics described in Section 3 of {{RFC3986}}. An empty authority section is not allowed. Userinfo is not supported in the authority section, and host/port separation with a colon character is not used. The query and fragment sections have no defined semantics and are reserved for future use.

The path section can be used to reference a specific resource controlled by the account authority. A common use case is to reference an individual data record from the account's public data repository:

~~~
"at://" ACCOUNT-AUTHORITY "/" COLLECTION "/" RECORD-KEY
~~~

The collection part indicates the data record type (schema), and the record key identifies the individual record. The URI path section matches the path under which records are stored in the repository data structure described in {{AT-REPOSYNC}}.

# Syntax {#syntax}

The overall AT URI encoded string length limit is 8192 ASCII characters. AT URIs MUST NOT include a trailing slash.

The authority section of AT URIs can contain either a permanent account identifier or an account handle. The syntax of specific account identifier systems is out of scope for this document, but a few generic syntax restrictions apply to all such identifiers:

- the account identifier string is ASCII, containing letters (A-Z, a-z), digits (0-9), period (`.`), hyphen (`-`), underscore (`_`), and colon (`:`)
- other ASCII characters may be represented with percent encoding (percent character `%` followed by two hexadecimal characters)
- must not end in a colon (`:`)
- length (including percent encoding) is between 1 and 2048 characters
- MUST NOT be a simple DNS hostname (in other words, must not be an account handle)

The account handle system is out of scope for this document, but the handle syntax is as follows:

- at most 253 ASCII characters in total
- consists of multiple segments separated by periods (`.`)
- empty segments or leading and trailing periods are not allowed
- there must be at least two segments, and thus at least one period ("bare" top-level domains are not allowed)
- each segment must have between 1 and 63 characters, consisting of lower-case letters (a-z), digits (0-9), and hyphens (`-`)
- segments cannot begin or end with hyphens
- the last segment must not start with a digit

Handles MUST be normalized to lower-case when included in AT URIs.

## Record References {#record-ref}

An AT URI referencing a record has additional syntax restrictions.

The first path component must be a valid Namespace Identifier (NSID) string, as defined in Appendix D of {{AT-REPOSYNC}}. A non-normative summary of that syntax is:

- at most 317 ASCII characters in length
- overall NSID is case-sensitive
- "domain authority" part (reverse-order DNS hostname, with at least two segments separated by periods) separated from a final "name" part by a period (`.`)
- domain authority segments are each between 1 and 63 characters; consist of ASCII lower-case letters (a-z), digits (0-9), and hyphens (`-`); must not start or end with a hyphen; the first segment must not start with a digit
- the final name part is between 1 and 63 characters; consists of ASCII alphanumerics (A-Z, a-z, 0-9); must not start with a digit

The second path component is a Record Key string, with syntax defined in Section 3.1 of {{AT-REPOSYNC}}. A non-normative summary of that syntax is:

- length between 1 and 512 ASCII characters
- consists of alphanumerics (A-Z, a-z, 0-9), period (`.`), hyphen (`-`), underscore (`_`), colon (`:`), and tilde (`~`)
- case-sensitive
- literal values `.` and `..` are forbidden

# Examples {#examples}

The following are valid AT URIs referencing an account identifier:

~~~
at://did:plc:foxkcdp2jhdxd75z7uuqu3s2
at://handle.example.com
~~~

The following are invalid AT URIs under the generic syntax:

~~~
// trailing slash
at://did:plc:foxkcdp2jhdxd75z7uuqu3s2/

// userinfo
at://user:pass@did:plc:foxkcdp2jhdxd75z7uuqu3s2
~~~

The following are valid AT URIs referencing a record:

~~~
at://did:plc:foxkcdp2jhdxd75z7uuqu3s2/com.example.record/3mwp2ezf3fh22
at://did:plc:foxkcdp2jhdxd75z7uuqu3s2/com.example.more-sections.record/3mwp2ezf3fh22
at://did:plc:foxkcdp2jhdxd75z7uuqu3s2/com.example.otherRecordV2/3mwp2ezf3fh22
at://did:plc:foxkcdp2jhdxd75z7uuqu3s2/com.example.record/...
at://did:plc:foxkcdp2jhdxd75z7uuqu3s2/com.example.record/~home
at://did:plc:foxkcdp2jhdxd75z7uuqu3s2/com.example.r/1
~~~

The following are invalid record references (though they may be valid generic AT URIs):

~~~
// disallowed record keys
at://did:plc:foxkcdp2jhdxd75z7uuqu3s2/com.example.record/..
at://did:plc:foxkcdp2jhdxd75z7uuqu3s2/com.example.record/one@two

// disallowed NSIDs
at://did:plc:foxkcdp2jhdxd75z7uuqu3s2/example/3mwp2ezf3fh22
at://did:plc:foxkcdp2jhdxd75z7uuqu3s2/com.example/3mwp2ezf3fh22
at://did:plc:foxkcdp2jhdxd75z7uuqu3s2/com.EXAMPLE.record/3mwp2ezf3fh22
at://did:plc:foxkcdp2jhdxd75z7uuqu3s2/123.example.record/3mwp2ezf3fh22
at://did:plc:foxkcdp2jhdxd75z7uuqu3s2/com.example.bad-record/3mwp2ezf3fh22
at://did:plc:foxkcdp2jhdxd75z7uuqu3s2/com.example.record-/3mwp2ezf3fh22

// trailing slash
at://did:plc:foxkcdp2jhdxd75z7uuqu3s2/com.example.record/3mwp2ezf3fh22/

// query section
at://did:plc:foxkcdp2jhdxd75z7uuqu3s2/com.example.record/3mwp2ezf3fh22?key=value
~~~

# Security Considerations {#security}

Record references that use ephemeral account usernames (handles) instead of permanent account identifiers can have the authority of the reference change over time. Such references should be resolved to a permanent account identifier before being persisted to long-term storage. References stored in record data should always use permanent account identifiers.

# IANA Considerations {#iana}

## URI Scheme Registration

As noted in {{intro}}, the identifier syntax described in this version of the document is not eligible for registration in the IANA URI Registry under {{RFC7595}}.

If it were, registration metadata would be included in this section.

--- back

# Acknowledgments
{:numbered="false"}

This document is based on the original Authenticated Transfer URI design work by Paul Frazee and Daniel Holmgren, as described in {{ATPAPER}}.

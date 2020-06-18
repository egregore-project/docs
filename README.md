---
description: >-
  DRAFT. Proposing a software system that allows human cognition to drive
  change, and enables communication between software originally unintended to
  integrate.
---

# Egregore: A Personal Software System

## Introduction

Like the browser, the system is megalithic in that any application design is expressible.

The system is agnostic to implementation technology provided they fulfill the guarantees needed to serve its purpose and are externally compatible with other conforming systems.

## Identity

All identities for the purposes of the system are based on Public-key cryptography, which is conformable to a super-set for practically all identity systems in use today. This decision simplifies the operation of the system in offline scenarios and lends well to implementations that partition data across users.

The primary drawback of Public-key cryptography for identities is that because verification of signatures is performed by anyone, it is not possible to delete verification of a signature against a public key once it is obtained by another source. The system is capable of fulfilling user deletion requests, including removing public key records and signatures, but it cannot invalidate signatures already obtained.

Public-key cryptography operations used by the system are based on Daniel Bernstein's algorithms implemented in NaCL, and utilized by libraries like libsodium, due to their ease of use for developers and software community opinion of these implementations.

### User

A user is identified in the system by the public key of an Ed25519 key pair, or a key derived from a base high entropy key. All user-based operations are signed. One user is created on system initialization to act as the primary administrator, and may grant this status to other users or groups of users by appending the log with signed commands. 

### Roles

Access control is provided by commands that form shared keys between the system user and users granted to perform specific actions. Revocation of roles is performed by the grantor.

## Schema

The core of the system is a set of ontological descriptions, or schemas, that describe the relationships between data structures, and how those data structures evolve over time.

All schema logs typically begin with a namespace declaration.

| Log Entry | Revision |
| :--- | :--- |
| NAMESPACE MyApp | - |

{% hint style="info" %}
Namespaces are case insensitive. If no explicit namespace is provided before schemas are defined, the NAMESPACE is implied as "default"
{% endhint %}

The schema is stored in an ordered append-only log format. Each entry in the schema is considered an internal revision by its index in the log. 

The following example shows the creation of a simple application that manages customer orders.

| Log Entry | Revision |
| :--- | :--- |
| NAMESPACE MyApp | - |
| SCHEMA Order | - |
| PROPERTY Number OF TYPE INT32 REQUIRED | 1 |
| SCHEMA Customer | - |
| PROPERTY Orders OF TYPE Order MANY | 2 |
| ... | ... |

{% hint style="info" %}
Schemas are case insensitive. Schema changes are relative to the last declared namespace.
{% endhint %}

As data structures change over time, modifying existing schemas means appending additional entries to the log.

#### Value Hashing

For each logical change to a specific schema in the log, we can produce the "value hash" of the schema using a 128-bit non-cryptographic hashing function, such as Murmur Hash 3. The value hash is produced by concatenating all non-computed user-provided values, and omitting all system-derived values.

| Log Entry | Revision | &lt;Value Hash&gt; |
| :--- | :--- | :--- |
| SCHEMA Customer | - | - |
| PROPERTY Name OF TYPE TEXT DEFAULT 'Unknown' | 3 | &lt;Hash&gt; |

In the case above, a new property is added to the Customer schema. When a new property is added, a default value is required. This ensures that previous records are usable when the system retrieves them in the context of later revisions.

## Data Records

Data structures are captured in an append-only log format. They consist of variable-length entries describing the new record or changes to an existing record.

| UUID | Index | Value | &lt;Value Hash&gt; | &lt;Signature&gt; |
| :--- | :--- | :--- | :--- | :--- |
| &lt;NULL&gt; | 0 | 12345 | ORA | \*\*\* |

The entry above represents creating a new Order record, whose Number property is '12345'. The Index is relative to the property for the intended schema and its target version, in order of effective appearance in the log. This is all of the user-provided data, and what would form the basis for this new record's Value Hash. In addition to this, the entry also requires several system-derived data points, listed below.

#### Value Hashing

For each logical change to a specific data record in the log, we can produce the "value hash" of the record using a 128-bit non-cryptographic hashing function, such as Murmur Hash 3. The value hash is produced by concatenating all non-temporal user-provided values, and omitting all system-derived values.

| **Schema Hash** | Sequence | Revision | UUID | Timestamp | Owner | DeletedAt | DeletedBy |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **OB** | 1 | 1 | abc123 | ISO-8601\[IANA\] | TEXT | \(ISO-8601\[IANA\]\) | \(TEXT\) |

* **Schema** Hash is provided during the operation to create or update the data record, as it is necessary to define which data structure and version the changes are intended for.
* **Sequence** is used as an internal key for the originating record, and is assigned by the system.
* **UUID** is a 128-bit key used as an external key for the originating record, and is assigned by the system if it is not provided by the user. After creation, no other data record may have the same UUID unless it is a record that describes a change to an existing record.
* **Timestamp** is used for time-series operations, and is a combination of an ISO-8601 timestamp concatenated with an IANA time zone identifier.
* **Owner** is a system-determined identity for requesting record creation. If the entry is an update to an existing record, this value represents the identity that requested the update.
* **DeletedAt** is used for time-series operations, and is a combination of an ISO-8601 timestamp concatenated with an IANA time zone identifier. _This value is uninitialized or omitted if the record is not deleted._
* **DeletedBy** is a system-determined identity for requesting record deletion. _This value is uninitialized if the record is not deleted._

## Versioning

Schema versioning is performed using a command to associate a revision with an external named version.

| Log Entry | Revision | &lt;Value Hash&gt; | &lt;Signature&gt; |
| :--- | :--- | :--- | :--- |
| VERSION "One" _\(Revision\)_ | 12 | LB | \*\*\* |
| DEPRECATE VERSION "One" | 13 | LC | \*\*\* |

{% hint style="info" %}
Version names are case insensitive. If a specific revision is not provided in the command, then the last revision is used.
{% endhint %}

All external data record operations are traceable to a target version, or the default version, which is the most recent external version. If there are no external versions, or all external versions are deprecated, the system is in an unpublished state. Data operations are accessible by explicit revision number if granted.

#### Revision Sets

For the purposes of relationships between data structures, a revision set is the group of all schema as they existed at a particular revision. This means that a relationship between one schema is always defined as the relationship between schema versions prior to the relating schema. The value hash is used to identity schema in a given revision set, and schema versions appear in every subsequent revision set in which the schema is not the subject of change.


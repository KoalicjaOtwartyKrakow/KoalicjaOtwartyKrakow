- Start Date: 2022-03-10
- RFC PR: 
- SafeShelter Issue: https://safeshelter.youtrack.cloud/issue/RFC-2

# Summary

This is an RFC to define audit trail for the system.

# Motivation

For audit and transparency into the process purposes, the system cannot just overwrite some data. Organization Admins need to be able to see the history of *Guest* assignments to *Accommodation Units* or which *Team Member* worked on which *Guest* *Entry*.

# Detailed Design

## Use of IETF Keywords

This document employs a subset of the Internet Engineering Task Force keywords found in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119). These words are MUST, SHALL, SHOULD, MAY and their counterparts MUST NOT, SHOULD NOT, MAY NOT. They are capitalized throughout the document to draw attention to their special status as keywords used to indicate requirements levels.

Readers are directed to interpret them as requirements at levels consistent with their term of art definitions in the IETF [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Business Architecture

### Capabilities

Provide a way to track important state changes to the application's data, without the need to parse the logs.

### Assumptions

1. Functionality SHALL not be developed from scratch. An existing solution SHALL be used.

### Dependencies

There are available off-the-shelf FOSS solutions out there:
- [pgMemento](https://github.com/pgMemento/pgMemento) - it is bare PostgreSQL solution using triggers. It is able to track  DDL changes, but it is not necessary and maybe even problematic given Alembic migrations. Stores audit data as JSONB deltas.
- [SQLAlchemy-Continuum](https://github.com/kvesteri/SQLAlchemy-Continuum) - A proper SQLAlchemy extentions. Actively maintained. Supports Alembic migrations. Does not seem to support tracking WHO made a change. 78k downloads a month.
- [Audit Trigger 91plus](https://github.com/2ndQuadrant/audit-trigger) - Does not track `SELECT` or `DDL` changes, but it is not required. Not sure how it'll work with Alembic migrations.
- [PostgreSQL-Audit](https://github.com/kvesteri/postgresql-audit) - Seems like a next version of SQLAlchemy-Continuum. README claims it, and it's by the same developer. It has tho less downloads (6k vs 78k), less stars, less commits, and last change was done almost a year ago, compared to month ago in Continuum.

Recommendation is to go with [SQLAlchemy-Continuum](https://github.com/kvesteri/SQLAlchemy-Continuum). Since it -- nor apparently neither of the plugins -- offers functionality of logging *who* made a change, development of custom plugin MAY be necessary.


### Risks 

### Business Risks

Storing too much data in the separate audit table may create data breach issues, as well as make RODO-related activities like *right to be forgotten* more complex.

### Impacts

None.

## Data Architecture

A new table SHALL be created, tracking at least:
- who made a change
- when the change was made
- which entity was modified
- value before the change
- value after the change

## Application Architecture

### Scenarios

No impact.

### User Interface

No impact.

### Services

Endpoints SHALL track changes to Guest entity - including at least field `guest.claimed_by` and `guest.accommodation_unit_id`. 

#### API

No impact.

### Availability

No impact.

### Security

To mitigate the risk of data tampering of audit log, as well as data breach a separation of duties SHALL be implemented. The Services MUST NOT be able to read nor append to audit tables.

## Infrastructure Architecture

No impact.

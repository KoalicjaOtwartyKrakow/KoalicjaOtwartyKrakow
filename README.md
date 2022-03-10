# About this document

This document outlines the scope and required functionality for the _KOK:on_ project.

# Summary

In the wake of the Russian invasion of Ukraine, hundreds of thousands of people migrated to Poland. They have various needs, and one of them is a temporary place to live. The project is inteded to support backoffice processes of SalamLab's Crisis Point at Radziwiłłowska 3, Kraków.

Today, members and volunteers at Crisis Point manage the database in a Google Spreadsheet. The data is encrypted at rest, and only people with volunteering contract have access to sensitive PII information, but with scaling of the organization and increasing number of people working concurrently on the database, they are starting to see drawback in this approach.

To better streamline volunteers' workflow the need is for dedicated software that optimizes flow and display of information, and enforce validation rules for people offering help, before they're connected with people in need. This need would also benefit the workflow of other organizations managing crisis situations.


## Glossary of terms

**Host** — a person who offers accommodation for people in need.

**Guest** — a person who uses the accommodation provided by the Host.

**Team Member** — verified member of organization with escalated privileges that manages the process of connecting *Host* with *Guest* and ensures that all safety protocols are observed.

**Accommodation Unit** — a house/apartment/room owned by the *Host* and made available to *Guests*

## Use of IETF Keywords

This document employs a subset of the Internet Engineering Task Force keywords found in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119). These words are MUST, SHOULD, MAY and their counterparts MUST NOT, SHOULD NOT, MAY NOT. They are capitalized throughout the document to draw attention to their special status as keywords used to indicate requirements levels.

Readers are directed to interpret them as requirements at levels consistent with their term of art definitions in the IETF [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Scenarios

The system needs to support the following scenarios:

### As a Guest

In the beta version, there are no Guest scenarios.

## As a Volunteer 

* 

* I want to list Accommodation Units
    * Filter by the following criteria:
        * Type
        * Location
        * Capacity
        * …
    * See the occupancy status (as difference between the capacity and the number of occupants).
* I want to list and filter Guests (criteria?)
    * In this context, filter and sort the list of applicable Accommodation Units
* I want to assign an Accommodation Unit to a Guest
    * After picking a Guest from the list, see a list of available Accommodation Units that would fulfill requirements of a given Guest
    * See detailed information of the Guest side by side with the list of Accommodation Units
* I want to inspect a newly registered host and accommodation unit
    * See a checklist of subtasks
    * Mark subtasks as done (and/or provide additional comments?)
* I want to check in with a Guest after a given period of time
    * See a checklist of subtasks
    * Mark subtasks as done (and/or provide additional comments?)

## As a Host (Priority: 2)

In the beta version there are no Host scenarios.

## As a System Administrator

* I want to be able to import data from Salam Lab's legacy database in Google Spreadsheet.

## Assumptions

## Constraints

1. Applicable GDPR laws: https://gdpr-info.eu/

## Dependencies

### Identity verification

The system needs to be able to verify the identity of the *Host*. The project does not aim at providing solution to this problem as there are vendors in the market offering such solutions.

Considered options are:
- Blue Media 1pln money transfer loop, openbanking-based verification, "Selfie"-biometrics verification, Liveness check biometric verification.
- Authologic - banking KYC, 1gr money transfer loop, "Selfie" biometrics, email verification, polish cell number verification, foreign cell number verification, digital id w/ PIN, digital ID w/o PIN.
- Autenti

## Risks

### Business Risks

### Technical Risks

1. No methods of authentication are 100% foolproof. Dedicated agent, can manipulate all above protective measures.

### Impacts

No organizational, training, documentation or process impacts have been identified.

## Technology

### Frontend

- Node.js 14.x
- React

### Backend

- Python 3.9
- Google Cloud Functions Framework


### Database

- PostgreSQL 13

## Architecture

Application MUST be deployed to three separate environments: *Development*, *Staging* and *Production*.

Neither *Development* nor *Staging* environment MUST NOT include PII data. It SHOULD include synthetic data for realism and validation purposes.


![SalamLab Backoffice Architecture](./architektura.png)

The application interfaces with the following applications/systems:

1. **GitHub** - Application's Continuous Integration/Continuous Deployment process observes GitHub master branch, and development branch

## Security

1. GCP Secrets Manager service MUST be used to secure the User ID/passwords (Database, etc.). The credentials MUST be retrieved during startup but cached in RAM to avoid repeated dip into the Secrets Manager.
2. There MUST be RBAC (Role Based Access Controls) to Cloud SQL.
3. Cloud SQL MUST have internal IP address only. Connections to Cloud SQL MUST be done via *Bastion Host*.
4. There MUST be strong RBAC around Secrets.
5. There MUST be data encryption at rest in Cloud SQL for any sensitive information in *Production* environment.

## API

1. Backend MUST expose dedicated REST API for the frontend.
2. Backed MAY expose REST API for authenticated 3rd party users.
3. REST API MUST be defined using Swagger 2.0 specification. Version 2.0 is the highest OpenAPI version supported by [Cloud Endpoints](https://cloud.google.com/endpoints/docs/openapi)
4. REST API OpenAPI definition must be available in root folder of [backend's git repository](https://github.com/KoalicjaOtwartyKrakow/backend/blob/main/api.yaml)

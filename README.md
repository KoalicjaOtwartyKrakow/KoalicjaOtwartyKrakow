# About this document

This document outlines the scope and required functionality for the _KOK:on_ project.

# Summary

In the wake of the Russian invasion of Ukraine, hundreds of thousands of people migrated to Poland. They have various needs, and one of them is a temporary place to live. The project is inteded to support backoffice processes of SalamLab's Crisis Point at Radziwiłłowska 3, Kraków.

Today, members and volunteers at Crisis Point manage the database in a Google Spreadsheet. The data is encrypted at rest, and only people with volunteering contract have access to sensitive PII information, but with scaling of the organization and increasing number of people working concurrently on the database, they are starting to see drawback in this approach.

To better streamline volunteers' workflow the need is for dedicated software that optimizes flow and display of information, and enforce validation rules for people offering help, before they're connected with people in need. This need would also benefit the workflow of other organizations managing crisis situations.

## Use of IETF Keywords

This document employs a subset of the Internet Engineering Task Force keywords found in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119). These words are MUST, SHALL, SHOULD, MAY and their counterparts MUST NOT, SHOULD NOT, MAY NOT. They are capitalized throughout the document to draw attention to their special status as keywords used to indicate requirements levels.

Readers are directed to interpret them as requirements at levels consistent with their term of art definitions in the IETF [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Business Architecture

### Capabilities

Provide backoffice functionality for Radziwiłłowska 3 Crisis Point.

### Glossary of terms

* **Host** — a person who offers accommodation for people in need.

* **Guest** — a person or representative of a group (including men, women, children and pets) that uses the accommodation provided by the Host.

* **Team Member** — verified member of organization with escalated privileges that manages the process of connecting *Host* with *Guest* and ensures that all safety protocols are observed.

* **Accommodation Unit** — a house/apartment/room owned by the *Host* and made available to *Guests*

### Assumptions

1. System is not going to enforce strict validation rule for information stored, except for Personally Identifiable Information of both *Guest* and *Host* required to ensure safety of all parties.

### Constraints

1. Applicable GDPR laws: https://gdpr-info.eu/

### Dependencies

1. The system needs to be able to verify the identity of the *Host*. The project does not aim at providing solution to this problem as there are vendors in the market offering such solutions. \
\
Considered options are:
   - [Blue Media](https://bluemedia.pl/) - 1pln money transfer loop, openbanking-based verification, "Selfie"-biometrics verification, Liveness check biometric verification.
   - [Authologic](https://authologic.com/pl) - banking KYC, 1gr money transfer loop, "Selfie" biometrics, email verification, polish cell number verification, foreign cell number verification, digital id w/ PIN, digital ID w/o PIN.
   - [Autenti](https://autenti.com/pl/) - the offer seemed to include only e-signature verification, and didn't suit project needs.

## Risks

### Business Risks
1. Having large amount of PII data available to indivituals could lead to data leakage/loss. To mitigate this, access to backoffice system SHOULD be periodically audited by organization. To support this, backoffice system SHALL not introduce own AuthN user database, but SHALL use external directories managed by the organization.

### Technical Risks

1. No methods of authentication are 100% foolproof. Dedicated agent, can manipulate all above protective measures.

### Impacts

No organizational, training, documentation or process impacts have been identified.

## Technology

### Frontend

- Node.js 16.x LTS
- React

### Backend

- Python 3.9
- SqlAlchemy
- Google Cloud Functions Framework

### Database

- PostgreSQL 13

## Application Architecture

### Conceptual Diagram

![SalamLab Backoffice Concept Diagram](./koncepcja.png)

Description of steps:
1. Organization admin adds user to User Directory
2. Reception volunteer navigates to Guest Registration Form
   * a/ Web-based form is served
   * b/ When form is submitted, a REST API is called
   * c/ Data is stored in the database
3. Apartment Team Volunteer navigates to the Admin Panel
   * a/ Admin Panel forces user to authenticate with their organization
4. Volunteer verifies the data of *Guest* using Guest Module of Admin Panel
5. Volunteer verifies the data of *Accommodation* using Accommodation Module of Admin Panel
6. Volunteer verifies the data of *Host* using Host Module of Admin Panel
7. Volunteer navigates to *Guest* details, searches for the matching appartment and assigns it to the *Guest*

### Scenarios

The system needs to support the following scenarios:

#### As a Guest

In the beta version, there are no Guest scenarios.


#### As a Team Member

##### ... I want to access Admin Panel

Admin Panel's landing page SHALL provide means to navigate to three availalbe modules: *Guest*, *Host* and *Accommodation*. Selecting *Guest* module SHALL open a *Guest* listing page providing summary of all *Guests* in the system. Similarly, selecting *Host* or *Accommodation* modules SHALL open listing page of *Hosts* and *Accommodations* respectively.

##### ... I want to list all Guests

*Guests* listing page SHALL provide description of icon status indicators:
   - Meat-free diet - does *Guest* require meat-free diet
   - Food allergies - does *Guest* have any food allergies
   - Gluten-free diet - does *Guest* require gluten-free diet
   - Lactose-free diet - does *Guest* require lactose-free diet

*Guests* listing page SHALL provide summary information of all *Guests* in the system in a tabular form, including columns:
   - Full name - First Name and Last Name of the *Guest*.
   - Phone number - Including international and local prefixes
   - Status - Priority status of a Guest. This can be one of the following:
     * Does not respond - after initial registration, volunteers were unable to contact the *Guest*
     * Accommodation not needed - legacy, deprecated value, *Guest* were registered for another reason. Can be found in imported data.
     * En route in Ukraine - *Guest* is traveling, still in Ukraine
     * En route in Poland - *Guest* successfully crossed the border, and is traveling to Kraków
     * In Kraków - *Guest* is here
     * At Radziwiłłowska 3 - *Guest* is in Crisis Response Center. Immediate attention required.
     * Accommodation Found - *Guest* is fine for now. This can change tho.
     * Updated - legacy, deprecated value, can be found in imported data.
   - Priority - when accommodation is required
   - How many - How many people are in the group. This should be split into Total, Men, Women, and Children count.
   - How long - for how long a stay is required. Row order of magnitude.
   - Remarks - The field SHALL include both icons representing *Food Allergies*, *Meat-free diet*, *Gluten-free diet*, *Lactose-free diet* as well as free-form notes from the *Guest*.

If not specified otherwise, System SHALL by default return the list of *Guests* sorted by *Priority Status*, and *Priority Date*. If not specified otherwise, System SHALL use following order while sorting by *Priority Status*:
   - At Radziwiłłowska 3
   - Updated
   - In Kraków
   - En route in Poland
   - En route in Kraków
   - Does not respond
   - Accommodation Found
   - Accommodation Not Needed 
If not specified otherwise, System SHALL use ascending order while sorting by *Priority Date*. 

Upon selecting a row, a Guest Details screen MUST be shown.

##### ... I want to see Guest Details

System MUST allow user to see detailed view of selected *Guest* with all information. The information SHOULD be broken down in logical groups:
- Personal data:
  * Priority Status
  * Full Name
  * E-mail
  * Phone
- Stay Information:
  * Priority Date
  * Stay duration
  * Desired destination
- Number of people
  * Total number of people in the group
  * The number of men
  * The number of women
  * The number of children, with the age for each child.
- Additional information
  * Information if *Guest* has pets
  * Detailed free-form detailed information on pets
  * Financial status - if *Guest* can afford to pay any rent, and if so - how much
  * Document number - number of the Passport, Id, Birth Certificate - any document they used to cross the border
  * Information if *Guest* is an agent - in data coming from Reception Desk at Radziwiłłowska 3, each person is registered as separate *Guest*, and an *Agent* is a person that acts as a proxy for a group of individually registered people.
- Detailed information
  * Special needs
  * Dietary information - information about *Meet-free*, *Gluten-free* and/or *Lactose-free* diet
  * Food allergies - free-form text description of any other restrictions and allergies a *Guest* has

Detailed *Guest* view SHALL include Accommodation Assignment information.

##### I want to assign Guest to Accommodation Unit

*Guest* detail view shall provide means to search for available *Accommodation Units*. If *Accommodation Unit* is assigned to a *Guest* and Team Member attempts to search for other *Accommodation Units* an Application SHALL warn the Team Member that current *Accommodation Unit* MAY be removed.

When presenting the Team Member with *Accommodation Units* matching search criteria, the Application SHALL show:
- Full Address - including Street name, Building and Apartment numbers where applicable, Zip code, City and Voivodeship name
- *Host* contact details including:
    * Full Name
    * Email
    * Phone number
    * Preferred hours to contact
- Verification status
- Accommodation's vacancy information

When presenting list of *Accommodation Units* matching search criteria for the *Guest* the Application SHALL sort the results by:
- Verification Status - in order of Verified, Verification Pending, Rejected
- Vacancies Free

### User Interface

User Interface SHOULD be branded with logos of organizations involved in operating and developing the system:
- [Laboratorium Pokoju - Salam Lab](http://salamlab.pl)
- [UA in Kraków - Fundacja Instytut Polska-Ukraina](http://uainkrakow.pl)
- [Fundacja Zustricz](http://zustricz.pl)
- [Koalicja Otwarty Kraków](http://koalicjaotwartykrakow.pl)

### Services

#### API

Services MUST expose dedicated REST API for the User Interface. They MAY expose REST API for authenticated 3rd party users. All REST API endpoints MUST be defined using Swagger 2.0 specification. Version 2.0 is the highest OpenAPI version supported by [Cloud Endpoints](https://cloud.google.com/endpoints/docs/openapi)


### Deployment Diagram

![SalamLab Backoffice Architecture](./architektura.png)

The application interfaces with the following applications/systems:

1. **GitHub** - Application's Continuous Integration/Continuous Deployment process observes GitHub master branch, and development branch

### Availability

The system SHOULD have availability of 95%.

### Security

GCP Secrets Manager service MUST be used to secure the User ID/passwords (Database, etc.). The credentials MUST be retrieved during startup but cached in RAM to avoid repeated dip into the Secrets Manager. There MUST be RBAC (Role Based Access Controls) to Cloud SQL, and very strong RBAC around Secrets. Cloud SQL MUST have internal IP address only. Connections to Cloud SQL MUST be done via *Cloud Proxy* for increased security and session timeouts. There MUST be data encryption at rest in Cloud SQL for any sensitive information in *Production* environment.

## Data Architecture

1. All operational data SHALL be stored in Cloud SQL database
2. All reporting data MAY be stored in Cloud SQL or Cloud Storage at discretion of DevOps team

## Infrastructure Architecture

### Separation of Environments

1. Application MUST be deployed to three separate environments: *Development*, *Staging* and *Production*.
   2, Neither *Development* nor *Staging* environment MUST NOT include PII data. It SHOULD include synthetic data for realism and validation purposes.
2. Application components that handle PII data MUST be deployed in EU
3. External system which exchange PII data with Application must be configurable with EU region for data storage.

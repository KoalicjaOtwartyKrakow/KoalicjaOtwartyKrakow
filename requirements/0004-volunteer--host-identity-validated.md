- Start Date: 2022-03-22
- RFC PR: 
- SafeShelter Issue: https://safeshelter.youtrack.cloud/issue/RFC-8


# Summary

This is an RFC to expose public form for host self-registration, with identity validation functionality.

# Motivation

To be able to replace Salam Lab's Google Form, but also add a required level of protection against abuse, by requiring strong identity validation of *Hosts*.

# Detailed Design

## Use of IETF Keywords

This document employs a subset of the Internet Engineering Task Force keywords found in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119). These words are MUST, SHALL, SHOULD, MAY and their counterparts MUST NOT, SHOULD NOT, MAY NOT. They are capitalized throughout the document to draw attention to their special status as keywords used to indicate requirements levels.

Readers are directed to interpret them as requirements at levels consistent with their term of art definitions in the IETF [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Business Architecture

### Capabilities

Provide functionality for *Hosts* to self-register, and for the *Volunteer* to see if the *Host* of *Accommodation Unit* they are about to send the *Guests* too is a legitimate person.

### Assumptions

1. Most identity verification strategies take some time, and are asynchronous by nature

### Constraints

1. Applicable GDPR laws: https://gdpr-info.eu/

### Dependencies

The system needs to be able to verify the identity of the *Host*. The project SHALL not aim at providing solution to this problem as there are vendors in the market offering such solutions.

Considered options are:
   - [Blue Media](https://bluemedia.pl/) - 1pln money transfer loop, openbanking-based verification, "Selfie"-biometrics verification, Liveness check biometric verification.
   - [Authologic](https://authologic.com/pl) - banking KYC, 1gr money transfer loop, "Selfie" biometrics, email verification, polish cell number verification, foreign cell number verification, digital id w/ PIN, digital ID w/o PIN.
   - [Autenti](https://autenti.com/pl/) - the offer seemed to include only e-signature verification, and didn't suit project needs.
   - [Stripe](https://stripe.com/docs/identity/verification-sessions) - the world-famous [Ukraine Take Shelter](https://www.ukrainetakeshelter.com/) uses it. Not entirely sure what is their pricing strategy. Might be free, but it's highly unreadable. This option popped up after decision was made.

The system SHALL perform verification via [Authologic](https://authologic.com).

## Risks

### Business Risks

1. No methods of identity validation are 100% foolproof. We are using the state-of-the-art solutions, but an evil actor determined enough, can bypass _in person_ validation at the bank, let alone a remote one like ours.

### Technical Risks

1. Authologic Terms of Use allow up to 1 minute of downtime without notice.

### Impacts

None.

## Data Architecture

`HostVerificationSession` table SHALL be introduced, with following fields:
- `id` an UUIDv4 compatible GUID
- `host_id` - UUIDv4 compatible GUID of a *Host* we are verifying
- `accommodation_id` - UUIDv4 compatible GUID of an *AccommodationUnit* that was added with the combined form
- `conversation_id` - id of conversation on vendor side

## Application Architecture

### Scenarios

The system needs to support the following scenarios:

#### As a Host

##### ... I want to offer my place for Guests

System MUST allow unauthenticated user to navigate to a Registration Form.

Registration Form View SHALL allow for entering *Host* information, including:
- Contact information:
    * Full Name
    * E-mail
    * Phone
    * Call after 
    * Call before
- Additional Information:
    * Comments - additional information about themselves
    * Languages spoken

Registration Form View SHALL allow for entering *AccommodationUnit* information, including:
- Accommodation data:
    * Address - a street name with street and apartment number, where available.
    * City
    * Voivodeship
    * Zip code - also known as Postal Code.
- Additional Information:
    * Comments - comments about the apartment itself
- Vacancies Information
    * Total available places
- Detailed Information:
  - Pets present
  - Pets accepted
  - Disabled People Friendly
  - LGBT Friendly
  - Parking Place Available
  - Easy Ambulance Access.

Registration Form View SHALL provide a user with RODO notice on how personal data will be collected, stored and used.

Registration Form View SHALL provide a user with notice that registration requires identity verification.

Registration Form View SHALL include "Proceed with Registration" button. When users clicks on "Proceed With Registration" button System shall redirect user to Authologic for identity verification.

When redirecting user to Authologic for identity verification, system SHALL provide Authologic with link to Verification Status View.

##### ... I want to check my verification status

System MUST allow user to check their verification status. If verification was successful system SHALL show a thank-you message to the user. If verification is Pending or Rejected system SHALL show a generic wait message.


##### ... I want to recover my account

When user tries to register with existing phone number system SHALL show Account Recovery View.
Account Recovery View SHALL notify user that this option is not available and to call Crisis Point's Hot Line.

### User Interface

No impact.

#### API

New endpoints SHALL be created for this feature. These endpoints SHALL be available without authentication.

### Services

#### New Host Registration Service (POST /registration)

If phone number sent in POST data already exists in database system SHALL return HTTP 302 and re-render generic Account Recovery View.

If phone number sent in POST data does not exist in database, System:
- SHALL create *Host* entry using data from POST, and set its status to Verification Pending.
- SHALL create *AccommodationUnit* entry using data from POST, and set its status to Verification Pending.
- SHALL create *VerificationSession* entry referencing newly created *Host* and *AccommodationUnit*.
- SHALL use newly created *VerificationSession*'s GUID to form `callback` pointing at `POST /registration/{GUID}` endpoint.
- SHALL use newly created *Host*'s GUID to form `returnUrl` pointing at Host Verification Status View.

#### Host Identity Verification Service (POST /registration/{verificationSessionId})

System SHALL only accept valid requests. Sender verification per [Authologic documentation](https://sandbox.authologic.com/docs/api/pl/v1.1/api/callback.html#weryfikacja-nadawcy) SHALL be determined as follows:
- `X-Signature-Timestamp` SHALL be no more than 5 minutes different from server time when last byte of message was received.
- `HMAC_SHA_256` hash of `<X-Signature-Timestamp>:<HTTP Response Body>` using secret signature key SHALL be equal to `X-Signature` header. When calculating the hash raw HTTP Body exactly as in Response SHALL be used, without any reformatting that might be introduced by backend frameworks.

When valid response is received from Authologic, system SHALL examine `result.identity.status` field of the response. System SHALL set verification status of both *Host* and *AccommodationUnit* correlated with given *VerificationSession* to:
- Verified if received status is `FINISHED` 
- Rejected if received status is `FAILED`
- Verification pending if received status is `IN_PROGRESS` or `PARTIAL`

When *Host* verification status is changed to Verified, system SHALL send and email to the given email address notifying a *Host* of the fact.

### Availability

No impact.

### Security

Callback endpoint SHALL verify authenticity of the request with HMAC_SHA_256 checksum using a dedicated signature key.

Registration endpoint SHALL always return HTTP 302 status code to protect from brute-force account enumeration attacks.

## Infrastructure Architecture

Signature Key needs to be stored as secret, outside of Terraform scripts.

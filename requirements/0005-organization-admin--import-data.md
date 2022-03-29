- Start Date: 2022-03-01
- RFC PR: None
- KOK:on Issue: https://github.com/KoalicjaOtwartyKrakow/KoalicjaOtwartyKrakow/issues/3


# Summary

This is an RFC to define data import from SalamLab's legacy Google Spreadsheets.

# Motivation

To be able to make a switch from legacy Google Spreadsheets based workflow to the KOK:on application, we need to be able to keep the data Volunteers have been working on.

# Detailed Design

## Use of IETF Keywords

This document employs a subset of the Internet Engineering Task Force keywords found in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119). These words are MUST, SHALL, SHOULD, MAY and their counterparts MUST NOT, SHOULD NOT, MAY NOT. They are capitalized throughout the document to draw attention to their special status as keywords used to indicate requirements levels.

Readers are directed to interpret them as requirements at levels consistent with their term of art definitions in the IETF [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Business Architecture

### Capabilities

Provide functionality to import data from Google Spreadsheets into database.


### Assumptions

1. Only *Guest*, *Hosts* and *AccommodationUnits* are subject to import. Other types of help gathered through the Google Form are either no longer in use, or negligible. 
2. Data will be automatically imported on the day of switchover from Google Spreadsheets to KOK:on. System SHALL not provide means for automatic incremental imports. All data entered into old spreadsheets after switchover SHALL be manually added to the system.

### Constraints

1. Applicable GDPR laws: https://gdpr-info.eu/

### Dependencies

None.

## Risks

### Business Risks
1. Imported data suffers from data integrity issues. Manual verification by volunteers will be required.

### Technical Risks

1. System MAY not be able to convert some data one-to-one.

### Impacts

1. Team Members will need to be trained which data may need to bo validated via phone or email after the import.

## Data Architecture

`Guests`, `Hosts` and `AccommodationUnits` tables shall have a new text field `system_comments` where data will be placed as free text in case system cannot convert the data into the target form.

Table `Hosts` SHALL have new fields:
- `for_how_long` - `\d+(d|w|m|y)`

Data SHALL be imported from two main Google Spreadsheets: `[salam] Mieszkania (Polska)` and `[salam] Ludzi (Ukrainska)`.

Mapping of tab `Automated Mieszkania` in `[salam] Mieszkania (Polska)` SHALL be as follows:
- Column `A` contents SHALL be appended to `acommodation_units.system_comments`
- Column `B` *NASZE UWAGI* contents SHALL be appended to `acommodation_units.system_comments`
- Column `C` *Na jak długo?*:
  - Values matching a pattern `\d+\s?(d|w|m|y)` SHALL be placed in `acommodation_units.for_how_long`
  - Other values SHALL be appended to `acommodation_units.system_comments`
- Column `D` *KIM jest juz zajety? (ID czlowieka)* contents shall be appended to `acommodation_units.system_comments`
- Column `F` *STATUS* contents SHALL be appended to `acommodation_units.system_comments`
- Column `G` *Timestamp* shall be converted to UNIX timestamp and placed in `acommodation_units.created_at` and `hosts.created_at`
- Column `H` SHALL be ignored.
- Column `I` *Miasto lokalu* contents SHALL be placed in `acommodation_units.city`
- Column `J` *Adres lokalu* contents SHALL be placed in `accommodation_units.address_line`
- Column `K` *Imię i nazwisko* contents SHALL be placed in `hosts.full_name`
- Column `L` *Adres email:* contents SHALL be placed in `hosts.email`
- Column `M` *Numer telefonu* contents SHALL be placed in `hosts.phone_number`
- Column `N` *Numer telefonu / adres email* contents SHALL be ignored
- Column `O` *Maksymalna ilość ludzi, które możesz przyjąć w swoim lokalu mieszkalnym?*:
  - Values matching a pattern `\d-\d` SHALL take an upper value and be placed in `accommodation_units.vacancies_total`
  - Other values SHALL be appended to `accommodation_units.system_comments`
- Column `P` *Uwagi: Czy posiadasz zwierzęta? Czy akceptujesz gości ze zwierzętami? Godziny w których możemy do Ciebie dzwonić? Masz inne uwagi?* contents SHALL be appended to `accommodation_unit.owner_comments`
- Column `R` *Jakie znasz języki?* SHALL be matched against languages *Polski*, *Ukraiński*, *Rosyjski*, *Angielski* and appropriate entries SHALL be added to `host_languages` table. Regardless of the matching, contents of the column SHALL be appended to `host.system_comments`
- Column `W` *Jeśli masz jakieś pytania, uwagi lub dodatkowe komentarze, na które nie znalazł_ś miejsca w formularzu, napisz je proszę poniżej:* contents SHALL be appended to `accommodation_unit.owner_comments`.

Contents of columns appended to `system_comments` SHALL be properly annotated with a name of the column in Google Spreadsheet, and a note that this is a value that didn't make it through the data import process.

## Application Architecture

### Scenarios

No impact.

### User Interface

No impact.

### Services

No impact.

#### API

No impact.

### Availability

No impact.

### Security

For DEV environment, following fields SHALL be sanitized with generated data:
- `hosts.full_name`
- `hosts.email`
- `hosts.phone_number`
- `accommodation_unit.city`
- `accommodation_unit.zip`
- `accommodation_unit.address_line`
- `guest.full_name`
- `guest.phone_number`


## Infrastructure Architecture

No impcat.

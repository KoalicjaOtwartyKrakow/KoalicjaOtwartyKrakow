- Start Date: 2022-03-26
- RFC PR: https://github.com/KoalicjaOtwartyKrakow/KoalicjaOtwartyKrakow/pull/4
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

Table `AccommodationUnit` SHALL have following changes:
- Field `status` SHALL be renamed to `verification_status`
- New field `workflow_status` SHALL be created as enum with following values:
  - Available 
  - Needs verification - The *AccommodationUnit* information might be outdated. *TeamMember* SHOULD call a *Host* to verify the status and information.
  - Withdrawn - The *Host* has withdrawn the consent.
  - Done - The *AccommodationUnit* is considered filled up. Moving to this status SHALL not be automated, because *AccommodationUnit* may have free vacancies that are technically useless (i.e. 2 beds were for children, even tho technically there is 1 bed left, but no other *Guest* may be assigned there), and *AccommodationUnit* should be removed from the pool even tho there are some vacancies left.

Data SHALL be imported from two main Google Spreadsheets: `[salam] Mieszkania (Polska)` and `[salam] Ludzi (Ukrainska)`.

Mapping of tab `Automated Mieszkania` in `[salam] Mieszkania (Polska)` SHALL be as follows:
- Column `A` contents SHALL be appended to `acommodation_units.system_comments`
- Column `B` *NASZE UWAGI* contents SHALL be appended to `acommodation_units.system_comments`
- Column `C` *Na jak długo?*:
  - Values matching a pattern `\d+\s?(d|w|m|y)` SHALL be placed in `acommodation_units.for_how_long`
  - Other values SHALL be appended to `acommodation_units.system_comments`
- Column `D` *KIM jest juz zajety? (ID czlowieka)* contents SHALL be used to match appropriate *Guest*. Generated UUIDv4 GUID for current *AccommodationUnit* SHALL  be placed in `guest.accommodation_unit_id` of a *Guest* with matching Column A value.
- Column `E` *WOLONTARIUSZ (kto się kontaktował)* contents will be used to set `workflow_status` to either `Available` if not empty, and `Needs verification` otherwise.
- Column `F` *STATUS* contents SHALL be converted to `workflow_status` such that:
  - `OUT` will be converted to `Withdrawn`
  - `DONE` will be converted to `Done`
  - `AVAILABLE` and empty will be converted to `Available`
  - `HOLD` will be converted to `Needs verification`
  - `CHANGED` will be converted to `Needs verification`
  - `Przjęcie krótkoterminowe` will be converted to `Available` if Column `E` is not empty, otherwise `Needs verification`
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

Mapping of tab `Ludzi` in `[salam] Ludzi (Ukrainska)` SHALL be as follows:
- Column `A` contents SHALL be noted, and used to match current *Guest* with appropriate entry in Column "KIM jest juz zajety? (ID czlowieka)" of *AccommodationUnits*.
- Column `B` contents SHALL be ignored
- Column `C` *DATA* contents SHALL be converted into UNIX timestamp and placed into `guests.created_at`
- Column `D` *STATUS* contents SHALL be converted to `guests.priority_status`
- Column `E` *imie i nazwisko wolontariusza* contents SHALL be appended to `guests.system_comments`
- Column `F` *Imie i Nazwisko* contents SHALL be placed in `guests.full_name`
- Column `G` *Telefon (z kodem kierunkowym)* contents SHALL  be placed in `guests.phone_number`, such that:
  - whitespaces SHALL be trimed
  - digits SHALL be grouped in threes.
  - if phone number consist of more than 9 digits, prefix SHALL be extracted and formatted with `+`.
- Column `H` *Priorytet* contents SHALL be converted into UNIX timestamp and placed in `guests.priority_date`
- Column `I` *TOTAL Ilość osób* contents SHALL be placed in `guests.people_in_group`
- Column `J` *Dorosly* contents SHALL be ignored, as there is no reliable way to decide how many women and men are there.
- Column `K` *Dzieci* contents SHALL be converted into an array with the number of elements specified by the value in the cell. The array SHALL have all `0`s, as there is no realiable way to extract the age of the children.
- Column `L` *Wiek Dzieci np 6m; 5 i 8* contents SHALL be placed in `guests.system_comments`
- Column `M` *Zwierzę*:
  - Value *Nie* SHALL set `guests.have_pets` to `False`
  - Other values SHALL set `guests.have_pets` to `True`, and append the contents of the cell to `guests.pets_description`
- Column `N` *Uwagi(Czy są kobieta w ciazy, inwalidy (jaka grupa?) Czy sa potrzebny leki, przelewanie krwi dializ etd? Czy są jakies alergie?* contents SHALL be placed in `guests.special_needs`
- Column `O` *Finanse* contents SHALL be placed in `guests.finance_status`
- Column `P` *Na ile Czasu*:
  - Values matching a pattern `\d+\s?(d|w|m|y)` SHALL be placed in `guests.how_long_to_stay`
  - Other values SHALL be appended to `guests.system_comments`
- Column `Q` *Uwagi* contents SHALL be placed in `guests.staff_comments`

For entries in tab `UPD Data from Reception` in `[salam] Ludzi (Ukrainska)` system SHALL import only those with value in Column `AA` *Czy osoba potrzebuje mieszkania? | Чи особа шукає житло?* equal to `Tak | Так`

Mapping of tab `UPD Data from Reception` in `[salam] Ludzi (Ukrainska)` SHALL be as follows:
- Column `A` contents SHALL be ignored.
- Column `B` *Notatki* contents SHALL be ignored.
- Column `C` *Timestamp* contents SHALL be converted into UNIX timestamp and placed in `guests.created_at`
- Column `D` *Imię | Ім'я* contents SHALL be appended to `guests.full_name`
- Column `E` *Nazwisko | Прізвище* contents SHALL be appended to `guests.full_name`
- Column `F` *Numer kontaktowy | Номер телефону* contents SHALL be placed in `guests.phone_number`
- Column `G` *Osoba do kontaktu? | Особа до контакту?* SHALL set `guests.is_agent` to `True` if equal to *Tak | Так* and to `False` otherwise.
- Column `H` *Numer dokumentu | Номер паспорту (ID)* contents SHALL be placed in `guests.document_number`
- Column `I` *Ile osób potrzebują zakwaterowania? | Скільки осіб потребують житла? [Mężczyźni | Чоловіки]* contents SHALL be placed in `guests.adult_male_count`
- Column `J` *Ile osób potrzebują zakwaterowania? | Скільки осіб потребують житла? [Kobiety | Жінки]* contents SHALL be placed in `guests.adult_female_count`
- Column `K` *Ile osób potrzebują zakwaterowania? | Скільки осіб потребують житла? [Dzieci | Діти]* contents SHALL be converted into an array with the number of elements specified by the value in the cell. The array SHALL have all `0`s, as the process of extraction children ages is not reliable
- Sum of values in Column `I`, Column `J` and Column `K` SHALL be placed in `guests.people_in_group`
- Column `L` *Wiek dziecka (dzieci) | Вік дитини (дітей)* contents SHALL be appended to `guests.system_comments`
- Column `M` *Wiek dorosłej osoby (osób) | Вік дорослої особи (осіб)* contents SHALL be appended to `guests.system_comments`
- Column `N` *Zwierzęta domowe | Домашні тварини [Pies | Собака]* contents if non-empty SHALL set `guests.have_pets` value to `True` and SHALL append *Pies* to `guests.pets_description`
- Column `O` *Zwierzęta domowe | Домашні тварини [Kot | Кіт]* contents if non-empty SHALL set `guests.have_pets` value to `True` and SHALL append *Kot* to `guests.system_comments`
- Column `Q` *Na jaki okres potrzebują zakwaterowania? | На який час потрібне житло?* contents SHALL take the upper limit value and convert it to `\d(d|w|m|y)` format and placed in `guests.how_long_to_stay`
- Column `R` *Mieszkanie płatne czy za darmo? | Житло платне чи безплатне?* contents SHALL be appended to `guests.finance_status`
- Column `S` *Jeśli z dopłatą to ile mogą wydać w miesiącu? | Якщо з доплатою то скільки можуть потратити за місяць?* contents SHALL be appended to `guests.finance_status`
- Column `T` *Czy mają własny transport? | Чи є власний транспорт?* contents SHALL be appended to `guests.system_comments`
- Column `U` *Miejscowość | Місцевість* contents SHALL be placed into `guests.desired_desitnation`
- Column `V` *Od jakiej daty  jest potrzebne zakwaterowanie? | Від якого числа (дати) потрібне житло?* contents SHALL be converted to UNIX timestamp and placed in `guests.priority_date`
- Column `W` *Notatki i komentarze dotyczące zwierząt domowych, zakwaterowania, stanu zdrowia, edukacji itd (opcjonalnie) | Нотатки і коментарі відносно домашніх тварин, житла, стану здоров"я, освіти та інше* contents SHALL be appended to `guests.staff_comments`
- Column `X` *Czy osoba zostaje na noc w naszym punkcie pomocy przy ul. Radziwiłłowska 3? | Чи особа залишається в нашому пункті допомоги на Radziwiłłowska 3?* contents SHALL be ignored
- Column `Y` *Numer telefonu (wolontariusza akceptującego zgłoszenie) | Номер телефону (волонтера, що приймає заявку)* contents SHALL be ignored.
- Column `Z` *Oświadczenie znajomości reguł przebywania | Підтвердження знайомості правил перебування* contents SHALL be ignored

Contents of columns appended to `system_comments` SHALL be properly annotated with a name of the column in Google Spreadsheet, and a note that this is a value that didn't make it through the data import process.

Contents appended to fields `staff_comments`, `system_comments` and `owner_comments` SHALL be appended using new line (`\n`) separator.


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

No impact.

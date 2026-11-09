# VCC Manager: manual cases on stage (no mocks)

Owner: Tamas Mate. Written 2026-11-09, against webbeds-connect master `9451fc16` (0.8.3-preview.1), Sabre `6fd566c`.
Source: SCN-15526 comments 287745, 288405, 288711 and the Test Cases field (C1 to C18, history).

These are **manual, Postman, stage only** cases. IDs `VCC-M-xx` are local to this list. They are not catalog IDs. The
**Covers** column maps each case to the catalog ID (`PULL:BK-xx`) or the proposed or battery ID it stands in for.

## Setup

- **Connector:** Sabre OBC stage, `{{baseUrl}}/sabre/sabre/.../obc/...?api-version=3.0`. Hotel 321004, rate plan `RAC`
  (refundable, takes no card). The Search, CheckAvailability and PreBook steps are the same as in T-01-D.
- **Token pairs:** `vccUniqueId` + `bookingCode` from the DOTW **stage** DB (query in documentation.md, section 6), or
  from Ruben. Put them in a Postman environment, **Current value only**: `vccToken`, `vccBookingCode`, and a second
  pair `vccToken2` / `vccBookingCode2` for VCC-M-15 and VCC-M-16.
- **V3 Book body:** as in T-01-D, with `"webBedsReference": "{{vccBookingCode}}"` and `"vccToken": "{{vccToken}}"`.
  Remove `CreditCardDetails`: V3 has no such field and ignores it. The Cancel uses the same `webBedsReference`.
- **Always** `metadata.includeSupplierAudit: true` on Book. The Book audit is **per room**, not at the root.
- **How to read the result:**
  - *Card reached Sabre*: the room's `supplierAudit` holds the Sabre **Initiate** request, and its `PaymentCard`
    has card type, number, expiry and CVV.
  - *No supplier call*: no Initiate or Commit in the audit, and no supplier reference.
  - *Category to HTTP status* (OBC V3): Validation 400, CredentialsNotValid 401, Payment 502, Timeout 504,
    Unexpected 500.
  - *Called vcc-manager or not*: this can't be seen on stage. The category tells you: `Validation` means it was
    refused before the call. `Payment` or `CredentialsNotValid` means vcc-manager answered.
- **Safety:** until SCN-15523, the audit shows the **real PAN and CVV**. Don't save, export or paste responses, and
  turn off Postman history. A case that books also cancels in the same pass. Never production.

## Cases

Format: ID | Given | When | Then | Pri | Env | Auto, and Covers.

### Happy path

| ID | Given | When | Then | Pri | Env | Auto | Covers |
|---|---|---|---|---|---|---|---|
| VCC-M-01 | V3 Book with `vccToken` and its own `bookingCode` as `webBedsReference`; refundable rate | Book is sent | 200, every room `Confirmed` with a supplier reference; the Initiate `PaymentCard` holds a Visa or MasterCard, a 13-19 digit number, an expiry that has not passed and a 3-digit CVV | P0 | Stage | Manual | PULL:BK-31 (stage), BK-26, OBC:BK-06, C1, C7 |
| VCC-M-02 | the booking from VCC-M-01 | Cancel is sent | 200, the room is cancelled; the Cancel needs no token | P0 | Stage | Manual | BK-26 |
| VCC-M-03 | the pair from VCC-M-01, already used once | VCC-M-01 is repeated (new Search and PreBook) | Confirmed again with the same card (last 4 and expiry match); then cancelled. Reading the card does not use the token up | P1 | Stage | Manual | C15 |
| VCC-M-04 | V3 Book with a token, **2 rooms** (T-04 rooms) | Book is sent | every room Confirmed; the same card is in every Initiate (the card is per booking, not per room); cancelled | P1 | Stage | Manual | BK-35 (partial: Sabre is Single, not PerRoom) |

### No token and backward compatibility

| ID | Given | When | Then | Pri | Env | Auto | Covers |
|---|---|---|---|---|---|---|---|
| VCC-M-05 | V3 Book with **no** `vccToken` field; any reference | Book is sent | Confirmed as before; no `PaymentCard` in the Initiate; cancelled | P0 | Stage | Manual | PULL:BK-29, C8 |
| VCC-M-06 | V3 Book with `"vccToken": ""` | Book is sent | same as VCC-M-05 (empty means no token, by design) | P1 | Stage | Manual | 288711 "to check", C8 |
| VCC-M-07 | V3 Book with `"vccToken": null` | Book is sent | same as VCC-M-05 | P2 | Stage | Manual | C8 |
| VCC-M-08 | your working **V1** Book, same body as before VCC | Book is sent | same answer and same supplier request as a V1 run from before 9451fc16 (same day, if you have one); cancelled | P0 | Stage | Manual | PULL:BK-30, C7 |
| VCC-M-09 | your working **V2** Book | Book is sent | as VCC-M-08 | P0 | Stage | Manual | PULL:BK-30, C7 |

### Refused by vcc-manager

| ID | Given | When | Then | Pri | Env | Auto | Covers |
|---|---|---|---|---|---|---|---|
| VCC-M-10 | V3 Book with an **invented** token (for example a new GUID) and a digit-only reference | Book is sent | 502, category `Payment`, error text carries code 2001; no supplier call; nothing booked | P0 | Stage | Manual | PULL:BK-32 (stage), C4 |
| VCC-M-11 | a real token with a **different** digit-only reference (for example `vccBookingCode2`) | Book is sent | 401, category `CredentialsNotValid`; no supplier call | P0 | Stage | Manual | PULL:BK-33 (stage), C3 |
| VCC-M-12 | a real token with **one letter changed in case**, its own reference | Book is sent | refused, category `Payment` or `CredentialsNotValid` (record which); no supplier call. The token is case sensitive | P2 | Stage | Manual | C17, B-03 |
| VCC-M-13 | the **oldest** pair in the DB (`ORDER BY DataUpdate ASC`) | Book is sent | Confirmed if the card has not expired; otherwise 502 `Payment` with no supplier call. Record which | P2 | Stage | Manual | C4 (expired), BK-31 proposed |

### Refused before vcc-manager (validation)

| ID | Given | When | Then | Pri | Env | Auto | Covers |
|---|---|---|---|---|---|---|---|
| VCC-M-14a | `vccToken` = `..` | Book is sent | 400, category `Validation`; no supplier call | P1 | Stage | Manual | BK-34 |
| VCC-M-14b | `vccToken` = `.` | Book is sent | as VCC-M-14a | P2 | Stage | Manual | battery |
| VCC-M-14c | `vccToken` of **65** characters | Book is sent | as VCC-M-14a | P1 | Stage | Manual | battery |
| VCC-M-14d | invented `vccToken` of exactly **64** characters | Book is sent | passes validation: 502 `Payment` (2001), like VCC-M-10 | P2 | Stage | Manual | battery (boundary) |
| VCC-M-14e | real token, reference `HYDRA12345678` (not digits) | Book is sent | 400 `Validation`; no supplier call | P1 | Stage | Manual | battery |
| VCC-M-14f | real token, reference `0000000` | Book is sent | 400 `Validation` | P2 | Stage | Manual | battery |
| VCC-M-14g | real token, reference of **21** digits | Book is sent | 400 `Validation` | P2 | Stage | Manual | battery |
| VCC-M-14h | real token, reference `{{vccBookingCode}}-1` (room reference) | Book is sent | 400 `Validation` | P2 | Stage | Manual | BK-35 |

### The card really makes a difference

| ID | Given | When | Then | Pri | Env | Auto | Covers |
|---|---|---|---|---|---|---|---|
| VCC-M-15 | a **refundable** Sabre rate that answers `242: Credit card required` without a card (not `RAC`; find one on 321001 or 321002); no token | Book is sent | fails with 242 (the control for VCC-M-16) | P1 | Stage | Manual | PULL:BK-13 |
| VCC-M-16 | the same rate as VCC-M-15, with a real token and its reference | Book is sent | Confirmed; card in `PaymentCard`; cancelled. This proves Sabre accepts the mapped card | P1 | Stage | Manual | PULL:BK-31 |
| VCC-M-17 | two V3 Books with **different pairs**, sent at the same moment (two Postman tabs, or the Runner) | both are sent | each Initiate carries its own card (different last 4); both cancelled | P1 | Stage | Manual | PULL:BK-35 |

### Card exposure

| ID | Given | When | Then | Pri | Env | Auto | Covers |
|---|---|---|---|---|---|---|---|
| VCC-M-18 | VCC-M-01 with `includeSupplierAudit: false` | Book is sent | no PAN, CVV or full token anywhere in the response | P0 | Stage | Manual | PULL:XC-11 |
| VCC-M-19 | VCC-M-01 with `includeSupplierAudit: true` | the response is read | **expected to fail until SCN-15523**: the PAN is visible in `supplierAudit`. Note only "PAN visible: yes/no", never the value | P0 | Stage | Manual | PULL:XC-11, XC-12 |

## Not testable manually on stage (mock or unit tests only)

Timeout and its retry (PULL:BK-34, C5); the 500 then success retry and a fresh nonce on the retry (BK-30); the stub's
400 for a bad signature or a stale timestamp (C2, C16); an expired card, a missing CVV or expiry, an Amex card, a
mismatched `VccUniqueId` (BK-31, BK-32, BK-33, BK-41); a redirect (BK-37); a connector without vcc-manager
configuration (BK-36, C10); PerRoom split with 2001 (BK-40); "only Book calls vcc-manager" (BK-38); logs (C6);
CMF Cancel (CHM:C-14); the library-only cases C9, C11 to C14 and C18.

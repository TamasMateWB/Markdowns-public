# VCC Manager: manual cases on stage (no mocks)

Owner: Tamas Mate. Written 2026-11-09, against webbeds-connect master `9451fc16` (0.8.3-preview.1), Sabre `6fd566c`.
Source: SCN-15526 comments 287745, 288405, 288711, the Test Cases field (C1 to C18, history) and BATTERY.md (9 Nov).
Updated 2026-11-09: battery cases added; the IDs in brackets under a case ID are the battery cases it includes.
**Run on Sabre stage on 2026-11-09: see [Results](#results-9-nov-2026).**

These are **Postman, stage only** cases, one folder each in the collection's **VCC Manager (SCN-15526)** folder. IDs `VCC-M-xx` are local to this list. They are not catalog IDs. The
**Covers** column maps each case to the catalog ID (`PULL:BK-xx`) or the Jira proposed / C ID it stands in for.
The catalog now runs to `PULL:BK-37` (BATTERY.md, 9 Nov): BK-36 is one card for every room, BK-37 is the Book on
stage with a real token.

## Setup

- **Connector:** Sabre OBC stage, `{{baseUrl}}/sabre/sabre/.../obc/...?api-version=3.0`. Hotel 321004, rate plan `RAC`
  (refundable, takes no card). The Search, CheckAvailability and PreBook steps are the same as in T-01-D.
- **Token pairs:** `vccUniqueId` + `bookingCode` from the DOTW **stage** DB (query in documentation.md, section 6), or
  from Ruben. They live in the environment **VCC stage pairs** (file `VCC stage pairs.postman_environment.json`), in
  `vccPairs`: a JSON list of `{token, ref}`. Each case folder takes a pair by its index (`C.vcc.pair`). A case whose
  pair is missing is skipped.
- **Running:** import the collection and the environment, select the environment, run the folder
  **VCC Manager (SCN-15526)** with the Collection Runner. Each case folder has all its inputs in `C` in the folder's
  pre-request script: copy a folder and change `C` to make a new case. Search clears the state of the previous case;
  Book is sent only after the PreBook safety gate (rate `Flexible`, no penalty within 24 h); Cancel cancels whatever
  Book booked. Results are masked `CASE|...` and `CANCEL|...` lines in the console and in the collection variable
  `runLog` (last 4 of the card only, never the PAN, CVV or token).
- **V3 Book body:** built by the Book pre-request script: `webBedsReference` is the pair's `ref`, `vccToken` the pair's
  `token`. V3 has no `CreditCardDetails` field.
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
| VCC-M-01<br>[B-54] | V3 Book with `vccToken` and its own `bookingCode` as `webBedsReference`; refundable rate | Book is sent | 200, every room `Confirmed` with a supplier reference; the Initiate `PaymentCard` holds a Visa or MasterCard, a 13-19 digit number, an expiry that has not passed and a 3-digit CVV | P0 | Stage | Postman | PULL:BK-37, BK-26, OBC:BK-06, C1, C7 |
| VCC-M-02 | the booking from VCC-M-01 | Cancel is sent | 200, the room is cancelled; the Cancel needs no token | P0 | Stage | Postman | BK-26 |
| VCC-M-03<br>[B-09] | the pair from VCC-M-01, already used once | VCC-M-01 is repeated (new Search and PreBook) | Confirmed again with the same card (last 4 and expiry match); then cancelled. Reading the card does not use the token up | P1 | Stage | Postman | C15 |
| VCC-M-04<br>[B-08, B-07] | V3 Book with a token, **2 rooms** (T-04 rooms) | Book is sent | every room Confirmed; the same card is in every Initiate (the card is per booking, not per room); cancelled | P1 | Stage | Postman | PULL:BK-36; BK-35 proposed (partial: Sabre is Single, not PerRoom) |
| VCC-M-20<br>[B-09] | a V3 Book with a token that was just confirmed (VCC-M-01) | the **same** Book request (same prebookToken and reference) is sent again at once | the second answer is either the same booking or a clear refusal; if Sabre books a second time, both carry the same card. Cancel **every** booking it made | P1 | Stage | Postman | — |
| VCC-M-21<br>[B-33] | VCC-M-01 and VCC-M-05 run back to back | the Book response times are compared | the Book with a token is slower by roughly one vcc-manager call (well under 1.5 s) and finishes inside `cutOffTimeMs`. Record both times | P2 | Stage | Postman | — |

### No token and backward compatibility

| ID | Given | When | Then | Pri | Env | Auto | Covers |
|---|---|---|---|---|---|---|---|
| VCC-M-05 | V3 Book with **no** `vccToken` field; any reference | Book is sent | Confirmed as before; no `PaymentCard` in the Initiate; cancelled | P0 | Stage | Postman | PULL:BK-29, C8 |
| VCC-M-06<br>[B-41] | V3 Book with `"vccToken": ""` | Book is sent | same as VCC-M-05 (empty means no token, by design) | P1 | Stage | Postman | 288711 "to check", C8 |
| VCC-M-07 | V3 Book with `"vccToken": null` | Book is sent | same as VCC-M-05 | P2 | Stage | Postman | C8 |
| VCC-M-08<br>[B-53] | your working **V1** Book, same body as before VCC (run it once with its dotwkms card and once without, if your request has one) | Book is sent | same answer and same supplier request as a V1 run from before 9451fc16 (same day, if you have one); with a dotwkms card, that card (not a vcc-manager one) reaches Sabre; cancelled | P0 | Stage | Manual | PULL:BK-30, C7 |
| VCC-M-09<br>[B-53] | your working **V2** Book, with and without a dotwkms card | Book is sent | as VCC-M-08 | P0 | Stage | Manual | PULL:BK-30, C7 |

### Refused by vcc-manager

| ID | Given | When | Then | Pri | Env | Auto | Covers |
|---|---|---|---|---|---|---|---|
| VCC-M-10 | V3 Book with an **invented** token (for example a new GUID) and a digit-only reference | Book is sent | 502, category `Payment`, error text carries code 2001; no supplier call; nothing booked | P0 | Stage | Postman | PULL:BK-32 (stage), C4 |
| VCC-M-11<br>[B-06] | a real token with a **different** digit-only reference (for example `vccBookingCode2`) | Book is sent | 401, category `CredentialsNotValid`; no supplier call | P0 | Stage | Postman | PULL:BK-33 (stage), C3 |
| VCC-M-12<br>[B-40] | a real token with **one letter changed in case**, its own reference | Book is sent | refused, category `Payment` or `CredentialsNotValid` (record which); no supplier call. The token is case sensitive | P2 | Stage | Postman | C17 |
| VCC-M-13<br>[B-13, B-14] | the **oldest** pair in the DB (`ORDER BY DataUpdate ASC`) | Book is sent | Confirmed if the card has not expired; otherwise 502 `Payment` with no supplier call. Record which | P2 | Stage | Postman | C4 (expired), BK-31 proposed |

### Refused before vcc-manager (validation)

| ID | Given | When | Then | Pri | Env | Auto | Covers |
|---|---|---|---|---|---|---|---|
| VCC-M-14a<br>[B-42] | `vccToken` = `..` | Book is sent | 400, category `Validation`; no supplier call | P1 | Stage | Postman | BK-34 proposed |
| VCC-M-14b<br>[B-42] | `vccToken` = `.` | Book is sent | as VCC-M-14a | P2 | Stage | Postman | battery |
| VCC-M-14c<br>[B-41] | `vccToken` of **65** characters | Book is sent | as VCC-M-14a | P1 | Stage | Postman | battery |
| VCC-M-14d<br>[B-41] | invented `vccToken` of exactly **64** characters | Book is sent | passes validation: 502 `Payment` (2001), like VCC-M-10 | P2 | Stage | Postman | battery (boundary) |
| VCC-M-14e<br>[B-44] | real token, reference `HYDRA12345678` (not digits) | Book is sent | 400 `Validation`; no supplier call | P1 | Stage | Postman | battery |
| VCC-M-14f<br>[B-44] | real token, reference `0000000` | Book is sent | 400 `Validation` | P2 | Stage | Postman | battery |
| VCC-M-14g<br>[B-44] | real token, reference of **21** digits | Book is sent | 400 `Validation` | P2 | Stage | Postman | battery |
| VCC-M-14h<br>[B-44, B-07] | real token, reference `{{vccBookingCode}}-1` (room reference) | Book is sent | 400 `Validation` | P2 | Stage | Postman | BK-35 proposed |
| VCC-M-14i<br>[B-41] | invented `vccToken` of **1** character (`a`) | Book is sent | passes validation: 502 `Payment` (2001) | P2 | Stage | Postman | — |
| VCC-M-14j<br>[B-42] | `vccToken` = `/`, then `%2F`, then `../view` (three sends) | Book is sent | 400 `Validation` or 502 `Payment` (2001); **never** a card, a 200 or a 500: the token must not change the vcc-manager URL. Record which | P1 | Stage | Postman | — |
| VCC-M-14k<br>[B-43] | `vccToken` with a space, a unicode letter (`é`), an emoji, and `\n` (four sends) | Book is sent | 400 or 502 as in VCC-M-14j; never a card, a 200 or a 500. Record which | P1 | Stage | Postman | — |
| VCC-M-14l<br>[B-44] | real token, reference `000` + `{{vccBookingCode}}` (leading zeros) | Book is sent | passes validation (leading zeros are allowed): record whether vcc-manager accepts it (Confirmed) or refuses it as another booking (401) | P2 | Stage | Postman | — |
| VCC-M-14m<br>[B-44] | real token, reference `SH12345` | Book is sent | 400 `Validation`. Note: a Sunhotels-style reference with letters can never carry a card | P2 | Stage | Postman | — |

### The card really makes a difference

| ID | Given | When | Then | Pri | Env | Auto | Covers |
|---|---|---|---|---|---|---|---|
| VCC-M-15<br>[B-55] | a **refundable** Sabre rate that answers `242: Credit card required` without a card (not `RAC`; find one on 321001 or 321002); no token | Book is sent | fails with a clear error carrying 242, not a 500 (the control for VCC-M-16). The rule for this case is still open: record what comes back | P1 | Stage | Postman | PULL:BK-13 |
| VCC-M-16<br>[B-54] | the same rate as VCC-M-15, with a real token and its reference | Book is sent | Confirmed; card in `PaymentCard`; cancelled. This proves Sabre accepts the mapped card | P1 | Stage | Postman | PULL:BK-37 |
| VCC-M-17<br>[B-01] | two V3 Books with **different pairs**, sent at the same moment (two Postman tabs, or the Runner) | both are sent | each Initiate carries its own card (different last 4); both cancelled | P1 | Stage | Postman | PULL:BK-35 |

### Card exposure

| ID | Given | When | Then | Pri | Env | Auto | Covers |
|---|---|---|---|---|---|---|---|
| VCC-M-18<br>[B-51] | VCC-M-01 with `includeSupplierAudit: false` | Book is sent | no PAN, CVV or full token anywhere in the response | P0 | Stage | Postman | PULL:XC-11 |
| VCC-M-19<br>[B-51] | VCC-M-01 with `includeSupplierAudit: true` | the response is read | **expected to fail until SCN-15523**: the PAN is visible in `supplierAudit`. Note only "PAN visible: yes/no", never the value | P0 | Stage | Postman | PULL:XC-11, XC-12 |
| VCC-M-22<br>[B-50, B-49] | the error answers of VCC-M-10, VCC-M-11 and VCC-M-14a | their error texts are read | no PAN or CVV; the token appears at most as `<VccToken>`, never in full. Record any full token you find | P1 | Stage | Postman | XC-12 proposed |

## Results, 9 Nov 2026

Sabre stage, hotel 321004 rate `RAC` (321001/321002 for VCC-M-15/16), check-in in 45 days, free cancellation until
22 Dec. Every Book passed the safety gate. **Every booking made was cancelled** (Cancel 200, rooms `Cancelled`).
Run through the browser with the same scripts as the collection. Card data below is masked: scheme, last 4 and expiry.

| ID | Verdict | What came back |
|---|---|---|
| T-01-D, T-01-S | Pass | Confirmed with no card (DOTW and Sunhotels), cancelled |
| VCC-M-01 | **Pass** | 200 Confirmed; Initiate `PaymentCard` MasterCard ****3280, 16 digits, expiry 10/29, CVV 3 digits |
| VCC-M-02 | Pass | Cancel 200, room `Cancelled` (the same for every booking below) |
| VCC-M-03 | Pass | the same pair again: Confirmed, the same card (****3280, 10/29) |
| VCC-M-04 | Pass | 2 rooms Confirmed; Sabre is Single, one Initiate, one card (Visa ****8630) |
| VCC-M-05, 06, 07 | Pass | no field, `""` and `null`: Confirmed, no `PaymentCard` |
| VCC-M-08, 09 | Not run | V1 and V2 requests are not in this collection |
| VCC-M-10 | Pass | 502 `Payment` "vcc-manager answered 500, code 2001"; no supplier call |
| VCC-M-11 | Pass | 401 `CredentialsNotValid` "vcc-manager answered 400, code 1000"; no supplier call |
| VCC-M-12 | Pass | 502 `Payment` (2001): vcc-manager treats a token in another case as unknown |
| VCC-M-13 | Pass | the oldest pair (1090511553) still holds a valid card (MasterCard ****6727, 11/29): Confirmed |
| VCC-M-14a, b, c | Pass | 400 `Validation` "The VccToken is not one vcc-manager issues" |
| VCC-M-14d, 14i | Pass | 64 characters and 1 character pass validation: 502 `Payment` (2001) |
| VCC-M-14e, f, g, h, m | Pass | 400 `Validation` "Booking number '…' is not the digits vcc-manager requires" |
| VCC-M-14j-1 (`/`) | **Fail** | 401 `CredentialsNotValid` (vcc-manager 400, code 1000). Not refused by validation: the token reached vcc-manager, and the answer is reported as a credentials problem. No card, nothing booked |
| VCC-M-14j-2 (`%2F`) | Pass | 502 `Payment` (2001) |
| VCC-M-14j-3 (`../view`) | **Fail** | as VCC-M-14j-1: 401 `CredentialsNotValid` (400, code 1000). No card, nothing booked |
| VCC-M-14k-1 to 4 | Pass | space, `é`, emoji, line break: 502 `Payment` (2001); never a card or a 500 |
| VCC-M-14l | Recorded | leading zeros pass validation, but vcc-manager refuses them as another booking: 401 `CredentialsNotValid` |
| VCC-M-15 | Pass | 502 `Provider` "242: Credit card required"; Sabre called, nothing booked. A clear error, not a 500 |
| VCC-M-16 | **Pass** | the same kind of rate with a token: Confirmed, MasterCard ****8627. Sabre accepts the mapped card |
| VCC-M-17 | Pass | the two Books sent at the same moment: Visa ****4219 and MasterCard ****1331, each its own |
| VCC-M-18 | Pass | audit off: no PAN and no token anywhere in the answer |
| VCC-M-19 | Fail, expected | audit on: the full PAN is in `supplierAudit` on every Book with a card (SCN-15523) |
| VCC-M-20 | **Finding** | the same Book sent twice made **two Sabre bookings**, both Confirmed with the same card (Visa ****1040); both cancelled. The card part passes (B-09); the duplicate booking is for Dev |
| VCC-M-21 | Pass | Book with a token 1.98 to 2.75 s, without 2.11 to 2.49 s: no visible cost. Refusals: validation 60 to 100 ms; vcc-manager 400 about 130 ms; 2001 about 250 ms (the retry) |
| VCC-M-22 | Pass | no error text carries the token, the PAN or the CVV: only "vcc-manager answered <status>, code <code>" |

**To raise with Dev:**

1. **`/` and `../view` in the token are not refused by validation** (VCC-M-14j). They reach vcc-manager and come back
   as `CredentialsNotValid`, the same answer as a wrong booking. Either refuse `/` and `..` segments as `Validation`
   before the call, or confirm the token is escaped as one path segment. No card leaked and nothing was booked.
2. **The same Book sent twice books twice on Sabre** (VCC-M-20), with the same reference and card. Not specific to
   VCC, but with a real card it means two charges.
3. Every error appears twice in the answer, at the root and on the room (seen on all the refusals). Cosmetic.
4. VCC-M-19 stays failed until SCN-15523.

## Not testable manually on stage (mock, config or unit tests only)

Timeout and its retry (PULL:BK-34, C5, B-25, B-30, B-31); 500 then success and a fresh nonce on the retry (BK-30,
B-28, B-29); the stub's 400 for a bad signature or a stale timestamp (C2, C16); an expired card, a missing CVV or
expiry, an Amex card, a mismatched `VccUniqueId` (BK-31 to BK-33, BK-41, B-03 to B-05, B-10 to B-12, B-16 to B-20);
odd answers and statuses (B-21 to B-24, B-26, B-27); a redirect (BK-37 proposed, B-23); wrong secret, ClientId case,
clock skew, replay (B-35 to B-39: direct probes to vcc-manager, not a Book); configuration (BK-36 proposed, C10,
B-45 to B-48); PerRoom split with 2001 (BK-40); "only Book calls vcc-manager" (BK-38); logs and `ToString` (C6, B-49,
B-52); cancel during the retry (B-34); B-02 (50 parallel Books needs 50 token pairs); Pull and CMF suppliers
(B-54 beyond Sabre, CHM:C-14); the library-only cases C9, C11 to C14 and C18.

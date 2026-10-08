---
type: decision
project: trillequipment
status: accepted
tags: [supabase, database, security]
date: 2026-10-07
---

# Supabase schema & access rules

Hub: [[Trill Equipment]] · How to run it: [[Supabase local dev]]

## Context
The live site (trillequipment.com) is a rental catalog with a reserve form: name, company, email, phone, equipment, start/end date, optional delivery address, terms checkbox. Staff call the customer back to confirm. Rates, deposit, phone and yard address are still `[PLACEHOLDERS]` on the site.

## Decisions

**Tables**
| Table | Holds |
|---|---|
| `equipment_types` | The 8 products on the site (5 trailers, 3 machines), rates, deposit, specs |
| `equipment_units` | Each physical trailer/machine I own |
| `reservations` | Reserve-form requests and their status |
| `reservation_documents` | License / insurance files per reservation |
| `site_settings` | Phone, yard address, deposit, cancellation notice, etc. |
| `staff_members` | Who can sign in, as `staff` or `admin` |

Reservation status: `pending → confirmed → out → returned` (or `cancelled` / `declined`).

**Access**
- Public visitors can only: read the catalog and public settings, submit a request, check availability. They can't read any reservations.
- Requests go through a database function (`submit_reservation_request`) instead of a direct insert. *Why:* it validates input (terms accepted, dates not in the past, end after start, max 5 requests/hour per email) and the public never touches the table directly.
- Public sign-up is off. Staff are invited. *Why:* customers don't need accounts; fewer accounts = less risk.
- Staff manage reservations; only admins change the catalog, settings or staff list.

**Integrity rules in the database**
- A unit can't be confirmed/out on two overlapping rentals.
- A unit can't be assigned to a booking for a different product type.
- Pending requests don't hold a unit; only confirmed/out do.

**Data**
- The catalog lives in a migration, not the seed file. *Why:* production needs it; seed data only loads locally.
- Unknown values (rates, deposit, capacity) are left blank rather than guessed.

**Storage**
- `equipment-images`: public, admins upload.
- `rental-documents`: private, staff only.

## Open questions
- Customer document upload from the site needs a small server function (not built).
- How should staff be notified of new requests? (`notification_email` setting reserved.)
- Real rates, deposit, cancellation notice days.

Repo commit: `376eb96` on `Joh's-Branch`.

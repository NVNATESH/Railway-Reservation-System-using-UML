# 📋 Use-Case Specifications — Railway Reservation System

> Companion document to the [README](../README.md) and the [OOAD Report](ooad-report.md).
> Specifications are derived from the use-case, sequence, activity and state-machine models in [`diagrams/`](../diagrams/).

Each of the five functional areas is specified below with: **participating actors**, the **primary use case**
(main + alternate flow), and a **summary table** of every remaining use case modelled in that area's
[use-case diagram](../diagrams/use-case/).

---

## UC-01 — User Registration

**Diagram:** [`diagrams/use-case/user-registration.mdj`](../diagrams/use-case/user-registration.mdj)
**Actors:** Passenger (primary) · Verification Service (supporting) · Railway Database Server (supporting)

### Primary use case: *Submit Registration*

| Field | Specification |
|-------|---------------|
| **ID / Name** | UC-01.08 — Submit Registration |
| **Brief** | A new customer registers for an account by providing profile details and credentials. |
| **Preconditions** | The applicant is not already registered. Registration portal is reachable. |
| **Postconditions** | A new `UserAccount` / `Passenger` record is persisted; registration is logged. |

**Main flow**

1. Applicant selects **Start Registration**.
2. System presents the registration form; applicant **Accepts Terms & Conditions**.
3. Applicant **Enters Account Details** (name, age, e-mail, phone, Aadhaar).
4. Applicant **Sets Credentials** (username / password).
5. Applicant **Submits Registration**.
6. System validates the profile format (via `UserProfileRules`).
7. System checks the username is not reserved (via `UsernameUniqueCheck`).
8. System validates password complexity (via `CredentialSecurity`).
9. System verifies mobile number via **OTP** and e-mail address via **verification link**.
10. System **Creates the User Account** and **Logs Registration Details** to the database.
11. System displays the success confirmation. *(final)*

**Alternate flows**

- **A1 — Aadhaar verification:** at start of registration, *Verify Aadhaar Proof* may be invoked through the Verification Service (modelled as an `«extend»` relationship on *Start Registration*).
- **A2 — Username already exists:** the username-reserved check fails → system shows an existing-user error and asks for a different username. *(cf. `diagrams/sequence/system/user-registration-v2.mdj`)*
- **A3 — Validation failure:** profile format or password complexity invalid → system re-displays the form with error messages.
- **A4 — Unverified contact:** OTP / e-mail link not confirmed → account creation is withheld until verification completes.

### Other use cases in this area

| Use case | Purpose |
|----------|---------|
| Start Registration | Entry point of the registration workflow |
| Enter Account Details | Capture profile data |
| Accept Terms & Conditions | Mandatory consent step |
| Set Credentails (Username/Password) | Capture login credentials |
| Verify Aadharr Proof | Identity proof verification (extends Start Registration) |
| Verify Moblie Number(OTP) | OTP channel verification (included) |
| Verify Email Address(Link) | E-mail channel verification (included) |
| Create User Account | Persist the account (included) |
| Log Registration Details | Audit-trail write (included) |

---

## UC-02 — Train Enquiry

**Diagram:** [`diagrams/use-case/train-enquiry.mdj`](../diagrams/use-case/train-enquiry.mdj)
**Actors:** Passenger (primary) · Railway Central Database Server (supporting)

### Primary use case: *Enter Search Criteria*

| Field | Specification |
|-------|---------------|
| **ID / Name** | UC-02.09 — Enter Search Criteria |
| **Brief** | The passenger searches for trains matching origin, destination and date. |
| **Preconditions** | Enquiry portal is accessible; schedule data is available. |
| **Postconditions** | Matching trains/schedules are displayed; the enquiry is logged. |

**Main flow**

1. Passenger **Accesses the Enquiry Portal**.
2. Passenger **Enters Search Criteria**.
3. Passenger **Selects Origin / Destination** stations (e.g. `DEL → BOM`).
4. Passenger **Specifies the Travel Date**.
5. System looks up valid trains — direct **and connected** routes (via `TrainDAO`).
6. If no direct train is found, system prompts for a **connected-route preference**.
7. System fetches detailed schedules and route stops and formats the results.
8. System displays the train list and schedules. *(final)*

**Alternate flows**

- **A1 — No train found:** system displays a *no-train-found* message. *(cf. `diagrams/activity/ticket-booking-flow.mdj`)*
- **A2 — Connected journeys:** passenger opts for a connected route (e.g. `DEL, connected BCT`); system calculates direct + connected combinations.

### Other use cases in this area

| Use case | Purpose |
|----------|---------|
| View Train Schedule | Browse schedules for a train |
| Check Seat Avaliability | Availability for a train/class/date |
| Find Fare and Quota Details | Fare and quota information |
| View Train Routes | Stops and route of a train |
| View Station Amenities | Amenities at a station (extends View Train Schedule) |
| Specify Travel Date | Part of search criteria |
| Select Origin / Destination | Part of search criteria |
| Access Enquiry Portal | Entry point |
| Log Enquiry Transactions | Audit trail (included by all enquiry use cases) |

---

## UC-03 — Ticket Booking

**Diagram:** [`diagrams/use-case/ticket-booking.mdj`](../diagrams/use-case/ticket-booking.mdj)
**Actors:** Passenger (primary) · Railway Central Database Server (supporting)

### Primary use case: *Initiate Ticket Payment* (booking completion)

| Field | Specification |
|-------|---------------|
| **ID / Name** | UC-03.10 — Initiate Ticket Payment |
| **Brief** | After journey selection and review, the passenger completes the booking by paying. |
| **Preconditions** | Trains searched; train & class selected; passenger information entered; fare calculated. |
| **Postconditions** | Seat reserved, PNR generated, e-ticket issued, transaction logged. |

**Main flow**

1. Passenger **Enters Journey Details** — **Selects Stations** and **Specifies Travel Date** (both included).
2. Passenger **Selects Train & Class**; system **Checks Current Berth Availability**.
3. Passenger **Adds Passenger Information** (traveler details, berth preference via `«extend»` on fare calculation).
4. System **Calculates the Final Fare**.
5. Passenger **Reviews the Booking Summary**.
6. Passenger **Initiates Ticket Payment** — system **Selects the Payment Mode**, **Authenticates Payment** and **Logs the Booking Transaction** (all included).
7. Seat is held, PNR is generated, e-ticket is displayed. *(final — cf. state machine `Booking Initiated → Awaiting Payment → Confirmed`)*

**Alternate flows**

- **A1 — Waitlist:** berth unavailable → passenger is **waitlisted**; on chart preparation the booking either confirms (*berthAllotted*) or fails with refund (*notConfirmed*). *(cf. `diagrams/state-machine/ticket-booking.mdj`)*
- **A2 — Payment failure:** booking transitions to *Booking Failed*, seat is released, passenger is notified; retry is possible. *(cf. `diagrams/activity/ticket-booking-flow.mdj`)*
- **A3 — No train / regret:** system shows waitlist/regret message before payment.

### Other use cases in this area

| Use case | Purpose |
|----------|---------|
| Enter Journey Details | Journey search entry point |
| Select Stations | Included by Enter Journey Details |
| Specify Travel Date | Included by Enter Journey Details |
| Select Train & Class | Train and class choice |
| Check Current Berth Availability | Included by selection |
| Add passenger Information | Traveler details capture |
| Apply Berth Preference Quota | Extends fare calculation |
| Calculate Final Fare | Fare computation |
| Review Booking Summary | Pre-payment confirmation |
| Authenticate payment | Included in payment initiation |
| Select payment Mode | Included in payment initiation |
| Log Booking Transactions | Audit trail (included throughout) |

---

## UC-04 — Payment & Billing

**Diagram:** [`diagrams/use-case/payment-billing.mdj`](../diagrams/use-case/payment-billing.mdj)
**Actors:** Registered User / Passenger (primary) · Bank / Payment Gateway · Billing & Accounting System · Backend Database Server (supporting)

### Primary use case: *Initiate Payment Transaction*

| Field | Specification |
|-------|---------------|
| **ID / Name** | UC-04.05 — Initiate Payment Transaction |
| **Brief** | The passenger pays for a booking through a gateway; artifacts (e-ticket, invoice) are produced. |
| **Preconditions** | A booking exists with a pending amount (cf. `BillingController.fetchPendingAmount(bookingID)`). |
| **Postconditions** | `Payment` recorded with gateway transaction id; e-ticket & invoice generated; confirmation sent. |

**Main flow**

1. Passenger **Selects the Payment Method** (Razorpay / Stripe / Paytm adapter, cf. component diagram).
2. Passenger **Initiates the Payment Transaction**.
3. System requests payment details and the passenger **Provides Payment Confirmation**.
4. System **Authorizes the Payment** with the Bank / Payment Gateway.
5. Gateway **Processes the Transaction**.
6. System **Confirms the Transaction & Sends Artifacts** — e-ticket + invoice by e-mail. *(final — cf. state machine `Success`)*

**Alternate flows**

- **A1 — Authorization failure:** payment transitions to *Failed*; error details are displayed; the passenger may retry. *(cf. `diagrams/state-machine/payment.mdj`)*
- **A2 — Refund:** a successful payment that is later cancelled transitions through *Refund Initiated → Refunded*. *(cf. `diagrams/state-machine/payment.mdj`)*

### Other use cases in this area

| Use case | Purpose |
|----------|---------|
| Confirm Transaction & Send Artifacts | Terminal step incl. e-mail delivery |
| Send Invoice to Email | Included in artifact dispatch |
| Generate E-Ticket | Included in artifact dispatch |
| Finalize Billing | Billing closure |
| Generate Invoice / Bill | Billing document creation |
| Request Invoice / Bill | Passenger-initiated invoice request |
| View Payment History | Passenger reviews past payments |
| Select payment Method | Gateway/mode selection |
| Authorize Payment | Gateway authorization |
| Process Transaction | Gateway processing |
| Provide Payment Confirmation | Included in initiation |

---

## UC-05 — Ticket Cancellation

**Diagram:** [`diagrams/use-case/ticket-cancellation.mdj`](../diagrams/use-case/ticket-cancellation.mdj)
**Actors:** Passenger (primary) · Railway Central Database Server (supporting)

### Primary use case: *Process Refund*

| Field | Specification |
|-------|---------------|
| **ID / Name** | UC-05.05 — Process Refund |
| **Brief** | After a confirmed cancellation, the system computes and processes the refund. |
| **Preconditions** | Ticket identified and valid; identity verified; cancellation rules applied. |
| **Postconditions** | `Refund` record created with amount & penalty; ticket status updated; refund executed by the gateway. |

**Main flow**

1. Passenger **Enters the Ticket ID / PNR**.
2. System **Verifies Ticket Validity** (with **Determine PNR Status**) and **Verifies User Identity** (with **Authenticate Identity**).
3. System **Applies Cancellation Rules** and **Calculates Cancellation Charges**.
4. System **Calculates the Refund Amount** (`refundAmount = totalFare − penalty`).
5. System **Handles Partial Cancellation** when only some travelers are cancelled.
6. Passenger **Confirms the Cancellation Request**.
7. System **Processes the Refund** through the gateway and **Logs the Cancellation Transaction**. *(final)*

**Alternate flows**

- **A1 — Invalid / cancelled ticket:** validity check fails → error shown, process aborts.
- **A2 — Identity mismatch:** authentication fails → cancellation not permitted.
- **A3 — Rule-based rejection:** cancellation rules make the ticket non-refundable → regret message.

### Other use cases in this area

| Use case | Purpose |
|----------|---------|
| Enter Ticket ID | PNR capture (includes ticket validity check) |
| Authenticate Identity | Included identity check |
| Verify Ticket Validity | Ticket/PNR status check |
| Determine PNR Status | PNR lookup |
| Calculate Cancellation Charges | Rule-based charges |
| Apply Cancellation Rules | Rules engine step |
| Handle Partial Cancellation | Subset of travelers cancelled (included by Process Refund) |
| Handle Partial Request | Partial cancellation variant |
| Calculate Refund Amount | Included by Process Refund |
| Confirm Cancellation Request | Passenger confirmation |
| Confirm Cancellationent | Confirmation variant (as modelled) |
| Log Cancellation Transactions | Audit trail (included throughout) |

> **Note:** the model also contains a placeholder element (`UseCase3`) and duplicate confirmations in
> [`ticket-cancellation.mdj`](../diagrams/use-case/ticket-cancellation.mdj) — kept as-drawn; candidates for cleanup in a future revision.

---

## Traceability

Cross-references between these specifications and the behavioural models:

| Area | Sequence (high-level) | Sequence (system) | Detailed sequence | Activity | State machine | Communication |
|------|----------------------|-------------------|-------------------|----------|---------------|---------------|
| User Registration | [`user-registration-flow`](../diagrams/sequence/high-level/user-registration-flow.mdj) | [v1](../diagrams/sequence/system/user-registration-v1.mdj) · [v2](../diagrams/sequence/system/user-registration-v2.mdj) | — | — | — | [`user-registration-flow`](../diagrams/communication/user-registration-flow.mdj) |
| Train Enquiry | [`train-enquiry-flow`](../diagrams/sequence/high-level/train-enquiry-flow.mdj) | [`train-enquiry`](../diagrams/sequence/system/train-enquiry.mdj) | — | — | — | — |
| Ticket Booking | [`ticket-booking-flow`](../diagrams/sequence/high-level/ticket-booking-flow.mdj) | — | [`ticket-booking`](../diagrams/sequence/detailed/ticket-booking.mdj) | [`ticket-booking-flow`](../diagrams/activity/ticket-booking-flow.mdj) | [`ticket-booking`](../diagrams/state-machine/ticket-booking.mdj) | [`ticket-booking-flow`](../diagrams/communication/ticket-booking-flow.mdj) |
| Payment & Billing | [`payment-flow`](../diagrams/sequence/high-level/payment-flow.mdj) | [`payment`](../diagrams/sequence/system/payment.mdj) | — | [`payment-flow`](../diagrams/activity/payment-flow.mdj) | [`payment`](../diagrams/state-machine/payment.mdj) | — |
| Ticket Cancellation | [`ticket-cancellation-flow`](../diagrams/sequence/high-level/ticket-cancellation-flow.mdj) | [`ticket-cancellation`](../diagrams/sequence/system/ticket-cancellation.mdj) | [`ticket-cancellation-refund`](../diagrams/sequence/detailed/ticket-cancellation-refund.mdj) | — | — | — |

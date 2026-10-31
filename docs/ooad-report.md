# 📖 OOAD Report — Railway Reservation System

> Object-Oriented Analysis & Design documentation for the Railway Reservation System.
> Companion documents: [README](../README.md) (visual model tour) · [Use-Case Specifications](use-case-specifications.md).

---

## 1. Introduction

### 1.1 Purpose
This report documents the object-oriented analysis and design of an online **Railway Reservation
System** (RRS). The system allows passengers to register, enquire about trains, book tickets,
pay online and cancel tickets with refunds. The full design is expressed as 28 UML models
created in [StarUML](https://staruml.io) and stored in [`diagrams/`](../diagrams/).

### 1.2 Scope
The system covers five passenger-facing flows — **User Registration, Train Enquiry, Ticket
Booking, Payment & Billing, Ticket Cancellation** — plus the supporting administration concerns
modelled in the packages (security, notification, baggage, railway operations).

Out of scope: loyalty programs, agent/Tatkal quotas, live train telemetry.

### 1.3 Method
A use-case-driven process was followed:

1. **Analysis** — use-case models per flow → domain (conceptual) class model.
2. **System design** — system sequence diagrams (black-box events) → interaction diagrams → state behaviour for objects with rich lifecycles.
3. **Architecture** — package layering, component decomposition, deployment topology.

---

## 2. Requirements Summary

| ID | Requirement | Modelled by |
|----|-------------|-------------|
| FR-1 | New customers register with identity/contact verification (Aadhaar, e-mail link, mobile OTP) | UC-01 · registration sequences · activity (registration) |
| FR-2 | Passengers enquire about trains by origin/destination/date, including **connected routes** | UC-02 · enquiry sequences |
| FR-3 | Passengers book tickets for a train/class with berth preference and fare calculation | UC-03 · booking sequence/activity/state |
| FR-4 | The system supports waitlisting with automatic confirmation or refund-on-failure at chart preparation | State machine `ticket-booking.mdj` |
| FR-5 | Payments are processed through pluggable gateways (Razorpay/Stripe/Paytm) with e-ticket & invoice generation | UC-04 · payment component adapters |
| FR-6 | Passengers cancel tickets; refunds follow cancellation rules with per-passenger (partial) handling | UC-05 · cancellation sequences · state machine |
| FR-7 | Every significant transaction is logged (booking, payment, cancellation, enquiry audit trail) | included `Log …` use cases |

---

## 3. Domain Analysis

The domain model ([`diagrams/domain-model/domain-model.mdj`](../diagrams/domain-model/domain-model.mdj))
identifies ten conceptual classes. The design class diagram
([`diagrams/class-diagram/design-class-diagram.mdj`](../diagrams/class-diagram/design-class-diagram.mdj))
refines them with attributes, operations and multiplicities.

| Class | Key attributes | Key operations |
|-------|----------------|----------------|
| `UserAccount` | userId, username, passwordHash, role | verifyCredentials(), updateProfile() |
| `Passenger` | passengerId, name, age, email, phone, aadhaar | register(), updateProfile() |
| `Ticket` | pnr, bookingDate, travelDate, totalFare, status | issue(), cancel() |
| `TravelerDetails` | travelerId, name, age, assignedBerth | assignBerth() |
| `Schedule` | routeId, source, destination, departureTime | getStops() |
| `Train` | trainNo, trainName, type, totalCapacity | getAvailability() |
| `SeatInventory` | inventoryId, date, classType, availableSeats | holdSeat(), releaseSeat() |
| `Payment` | paymentId, amount, gatewayTxnId, status | process() |
| `PaymentGateway` | providerName, apiEndpoint | authorizePayment(), processRefund() |
| `Refund` | refundId, refundAmount, penalty, status | calculateRefund(), process() |

### 3.1 Key relationships

| Relationship | Multiplicity | Rationale |
|--------------|--------------|-----------|
| `UserAccount` — `Passenger` | 1 — 1 | Every account belongs to exactly one passenger profile |
| `Passenger` — `Ticket` | 1 — 0..\* | A passenger may hold many tickets |
| `Ticket` — `Payment` | 1 — 1 | A ticket is settled by one payment |
| `Ticket` — `TravelerDetails` | 1 — 1..\* | A ticket covers one or more travelers |
| `Ticket` — `Refund` | 1 — 0..1 | A ticket has at most one refund |
| `Payment` — `PaymentGateway` | 0..\* — 1 | Payments execute through a gateway |
| `Schedule` — `Train` | 1 — 1 | A schedule instance belongs to a train |
| `Train` — `SeatInventory` | 1 — 0..\* | Per-date, per-class seat inventory |

### 3.2 Design observations

- **`SeatInventory` is the concurrency hotspot** — `holdSeat()` / `releaseSeat()` encapsulate the
  race-prone seat allocation, keeping reservation atomicity inside one object.
- **`Refund` as a first-class object** — cancellation charges (`penalty`) and status transitions are
  captured explicitly rather than as a side effect of `Ticket.cancel()`.
- **`PaymentGateway` abstraction** — the gateway is an interface-like class; concrete providers are
  isolated in the component architecture (see §5.2).

---

## 4. Behavioural Design

### 4.1 Object interactions
- **High-level sequence diagrams** (one per flow, [`diagrams/sequence/high-level/`](../diagrams/sequence/high-level/))
  assign responsibilities to boundary–control–entity objects: `…Screen` (boundary) → `…Controller`
  (control) → `…DAO` / domain objects (entity). Example, ticket booking:
  `BookingScreen → BookingController → ReservationDAO` with `verifySeatAvailability`, `holdSeat`,
  `createBooking(pnr)`.
- **System sequence diagrams** ([`diagrams/sequence/system/`](../diagrams/sequence/system/)) fix the
  black-box contract per flow before object design; registration exists as two iterations
  (`v1`, `v2`) showing the refinement step.
- **Detailed sequence diagrams** ([`diagrams/sequence/detailed/`](../diagrams/sequence/detailed/))
  cover the two most complex flows end-to-end: ticket booking (32 messages, involving
  `Train`, `Payment`, `Ticket`) and cancellation & refund (25 messages, involving
  `RefundService`, `PaymentGateway`, `NotificationService`).

### 4.2 State behaviour
Two objects have lifecycles rich enough to warrant state machines:

| State machine | States | Notable transitions |
|---------------|--------|---------------------|
| [Ticket Booking](../diagrams/state-machine/ticket-booking.mdj) | Booking Initiated · Awaiting Payment · Confirmed · Cancelled · Waitlisted · Booking Failed | `seatsAvailable / holdSeat` · `berthAllotted [chart prepared] / issueTicket` (waitlist → confirmed) · `notConfirmed [chart prepared] / refundFare` |
| [Payment](../diagrams/state-machine/payment.mdj) | Payment Initiated · Awaiting Authorization · Processing · Success · Failed · Refund Initiated · Refunded | `authorizePayment(cardData)` · `authorizationFailed / recordTransaction` · retry from *Failed* |

### 4.3 Workflows & collaborations
- **Activity diagrams** model the two decision-heavy workflows: booking
  (with no-train-found and payment-failure branches) and payment (gateway redirect loop on
  failure, confirmation notification at the end).
- **Communication diagrams** show the booking collaboration (9 objects, 21 messages) and the
  registration collaboration (7 objects, 20 messages) with numbered message sequences.

---

## 5. Architectural Design

### 5.1 Package layering ([`diagrams/package/package-diagram.mdj`](../diagrams/package/package-diagram.mdj))
Nine packages: Passenger · Booking · Train · Baggage · Payment · Railway Operations · Database ·
Notification · Security management. Passenger management depends on Booking/Baggage/Payment;
all operational packages converge on Notification and Security management.

### 5.2 Component decomposition ([`diagrams/component/component-diagram.mdj`](../diagrams/component/component-diagram.mdj))
The Passenger Portal fronts four domain services (Booking, Train, Ticket, Payment) backed by a
Railway Database, a Notification Service (with Email Gateway), and **three swappable payment
gateway adapters** — Razorpay, Stripe, Paytm — demonstrating provider-agnostic payment handling (FR-5).

### 5.3 Deployment ([`diagrams/deployment/deployment-diagram.mdj`](../diagrams/deployment/deployment-diagram.mdj))
Passenger devices reach a Web Server, which fronts an Application Server (Tomcat / Spring Boot)
connected to the Database Server, Payment Gateway Server and Notification Gateway Server.

---

## 6. Traceability Matrix

| Flow | Use case | Domain | Class | Sequence HL | Sequence SYS | Sequence DET | Activity | State | Communication | Architecture |
|------|----------|:------:|:-----:|:----------:|:------------:|:------------:|:--------:|:-----:|:-------------:|:------------:|
| User Registration | [`user-registration`](../diagrams/use-case/user-registration.mdj) | ✔ | ✔ | [`HL`](../diagrams/sequence/high-level/user-registration-flow.mdj) | [v1](../diagrams/sequence/system/user-registration-v1.mdj) · [v2](../diagrams/sequence/system/user-registration-v2.mdj) | — | — | — | [`comm`](../diagrams/communication/user-registration-flow.mdj) | ✔ (pkg) |
| Train Enquiry | [`train-enquiry`](../diagrams/use-case/train-enquiry.mdj) | ✔ | ✔ | [`HL`](../diagrams/sequence/high-level/train-enquiry-flow.mdj) | [`SYS`](../diagrams/sequence/system/train-enquiry.mdj) | — | — | — | — | ✔ (comp/dep) |
| Ticket Booking | [`ticket-booking`](../diagrams/use-case/ticket-booking.mdj) | ✔ | ✔ | [`HL`](../diagrams/sequence/high-level/ticket-booking-flow.mdj) | — | [`DET`](../diagrams/sequence/detailed/ticket-booking.mdj) | [`act`](../diagrams/activity/ticket-booking-flow.mdj) | [`sm`](../diagrams/state-machine/ticket-booking.mdj) | [`comm`](../diagrams/communication/ticket-booking-flow.mdj) | ✔ (all) |
| Payment & Billing | [`payment-billing`](../diagrams/use-case/payment-billing.mdj) | ✔ | ✔ | [`HL`](../diagrams/sequence/high-level/payment-flow.mdj) | [`SYS`](../diagrams/sequence/system/payment.mdj) | — | [`act`](../diagrams/activity/payment-flow.mdj) | [`sm`](../diagrams/state-machine/payment.mdj) | — | ✔ (comp: gateway adapters) |
| Ticket Cancellation | [`ticket-cancellation`](../diagrams/use-case/ticket-cancellation.mdj) | ✔ | ✔ | [`HL`](../diagrams/sequence/high-level/ticket-cancellation-flow.mdj) | [`SYS`](../diagrams/sequence/system/ticket-cancellation.mdj) | [`DET`](../diagrams/sequence/detailed/ticket-cancellation-refund.mdj) | — | — | — | ✔ (pkg) |

*✔ = covered by the shared project-wide model (domain model, design class diagram, package,
component and deployment diagrams apply to all flows).*

---

## 7. Conclusion

The model set traces every functional requirement from use case to architecture: each flow has a
use-case diagram and interaction models, the two behaviour-critical objects (booking, payment) are
specified as state machines and workflows, and the architecture isolates volatile concerns
(payment providers, notification channels) behind stable abstractions. Possible next iterations:
implement the design as a Spring Boot service skeleton, and clean up the two placeholder elements
noted in the cancellation use-case model.

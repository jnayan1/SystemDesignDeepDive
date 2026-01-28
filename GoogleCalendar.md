# System Design: Google Calendar

This document outlines a scalable, highly available, and consistent backend architecture for a Google Calendar-style system, designed for a Senior Software Engineer interview context.

---

## 1. Functional Requirements

* **Event Management:** Users can create, edit, and delete events (Title, Time, Location).
* **Invitations & RSVP:** Organizers can invite guests; guests can respond (Yes/No/Maybe).
* **Calendar Views:** Support for daily/weekly/monthly views.
* **Free/Busy Search:** Ability to check the availability of multiple users to find a meeting slot.
* **Recurrence:** Support for repeating events (e.g., "Every Monday at 10 AM").

## 2. Non-Functional Requirements

* **Strong Consistency for Owners:** A user must see their own changes immediately (Read-Your-Own-Writes).
* **High Availability for Readers:** The calendar must be viewable even if some background services are lagging.
* **Low Latency:** < 200ms for viewing a week; < 500ms for availability checks of 10+ users.
* **Scalability:** Support 100M+ users and handle high-concurrency "Thundering Herds" (e.g., corporate-wide meeting updates).

---

## 3. Data Model (Entities)

We use a **Relational Database (PostgreSQL)** to handle complex relationships and ensure ACID compliance for event ownership.

* **User:** `user_id` (PK), email, timezone, settings.
* **Event:** `event_id` (PK), `organizer_id` (FK), start_time, end_time, rrule (Recurrence Rule), title, location, version (for OCC).
* **Invitation:** `event_id` (FK), `guest_id` (FK), rsvp_status, last_notified_at.
* *Note: We have 1 invitation record per user per event to avoid contention on the main Event record.*


* **EventException:** `event_id` (FK), `original_time`, `new_time`, is_deleted. (Handles one-off changes to a recurring series).

---

## 4. API Design

* `POST /v1/events`: Create an event and trigger guest invitations.
* `PATCH /v1/events/{id}`: Update event metadata or time.
* `GET /v1/calendar/view?start={ts}&end={ts}`: Fetch all expanded instances for a date range.
* `POST /v1/availability`: Query multiple `user_ids` for their free/busy blocks.
* `PATCH /v1/invitations/{event_id}/rsvp`: Guest responds to an invite.

---

## 5. High-Level Architecture

1. **API Gateway:** Handles Auth, Rate Limiting, and Request Routing.
2. **Calendar Service:** Core logic for CRUD and recurrence expansion.
3. **Fan-out Service:** Handles the heavy lifting of sending invites to 100s of guests.
4. **Notification Service:** Manages **WebSockets** or **SSE** connections to push real-time updates to clients.

---

## 6. Deep Dive

### Scaling via Sharding

We shard the database by **`user_id`**.

* **Benefit:** All of a user's primary calendar data lives on one shard, making "View My Week" queries extremely fast and locally consistent.
* **The Challenge:** Cross-shard invites. If User A (Shard 1) invites User B (Shard 2), we need a reliable way to update Shard 2.

### Consistency: The Transactional Outbox

To maintain strong consistency for the owner but high availability for the system, we avoid distributed transactions.

1. **Step 1:** The event is saved to User A's shard. In the same transaction, a message is written to a local `Outbox` table.
2. **Step 2:** A **Message Relay** reads the outbox and publishes it to **Kafka**.
3. **Step 3:** A worker listening to Kafka writes the invitation to User B's shard.
*This ensures eventual consistency for the guest without slowing down the organizer.*

### Availability: The Busy-Block Bitmask

Calculating availability by querying raw SQL across 20 users is too slow.

* **Redis Bitmask:** We store each user’s day as a 96-bit mask (15-min slots).
* **O(1) Check:** To find a free time, we perform a bitwise `OR` across the participants' masks in Redis. This returns the result in microseconds.

### Handling "Thundering Herds"

When a meeting with 1,000 guests is updated:

* **Delta Updates:** Don't tell the client "Refresh everything." Send the updated event data *inside* the WebSocket payload.
* **Client-Side Jitter:** If a refresh is mandatory, clients add a random delay (0-3s) before hitting the server to prevent a CPU spike on the DB.
* **Request Collapsing:** The server uses a cache or "SingleFlight" mechanism to ensure that 1,000 identical "Get Event" requests only result in one database query.

### Conflict Resolution (Contention)

We use **Optimistic Concurrency Control (OCC)**. Every `Event` record has a `version`.

* If two organizers edit the same event, the second one will have a version mismatch and receive a `409 Conflict`.
* Since RSVPs are stored in a separate `Invitation` table per user, guest responses never contend with each other or the main event record.

| Feature | Protocol/Consistency | Reasoning |
|---------|---------------------|-----------|
| View My Week | Strong Consistency | Users cannot tolerate "ghost" or missing events they just created. |
| RSVP Status | Eventual Consistency | Propagating a "Yes" to the organizer can happen via Kafka in the background. |
| Real-time Sync | SSE / WebSockets | Prevents expensive "Polling" and ensures multi-device harmony. |
| Availability | Eventual (Redis) | Performance is prioritized. A bitmask check in Redis is faster than a DB join. |
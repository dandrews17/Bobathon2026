# Inventory System — Spring Boot Plan

## Overview

Build a Spring Boot **3.x** REST API that manages a physical item inventory. Stock is **reserved immediately** when a request is submitted — `stockAmount` is decremented at request time, not at distribution. Each unit of stock maps to exactly one request (one request per unit). Admins can update item locations and distribute reserved items. When a request is marked ready for pickup, the system notifies the user via their preferred channel (email or SMS). Normal users can view the status of their own requests.

---

## Sub-Task 1 — Domain Model

**Intent:** Define the core JPA entities and enumerations that the entire system is built on.

**Expected Outcomes:**
- `User`, `Item`, `ItemRequest`, and `NotificationLog` entities exist as JPA-mapped classes
- `UserRole`, `RequestStatus`, and `NotificationPreference` enumerations are defined
- All foreign key relationships are mapped via JPA annotations

**Todo List:**
- [ ] Create `UserRole` enum: `ADMIN`, `USER`
- [ ] Create `NotificationPreference` enum: `EMAIL`, `SMS`
- [ ] Create `RequestStatus` enum: `PENDING`, `ACKNOWLEDGED`, `READY_FOR_PICKUP`, `DISTRIBUTED`
- [ ] Create `User` entity: id, name, email, phoneNumber, notificationPreference, role
- [ ] Create `Item` entity: id, type, stockAmount, location
- [ ] Create `ItemRequest` entity: id, requestedBy (FK → User), item (FK → Item), status, requestedAt, resolvedAt
- [ ] Create `NotificationLog` entity: id, user (FK → User), request (FK → ItemRequest), channel, message, sentAt

**Relevant Context:**
- All entities go in `src/main/java/.../model/`
- Use `@Enumerated(EnumType.STRING)` for all enum columns
- `ItemRequest.status` defaults to `PENDING` on creation

**Status:** [ ] pending

---

## Sub-Task 2 — Repositories

**Intent:** Provide Spring Data JPA repositories so the service layer can query and persist entities without writing SQL.

**Expected Outcomes:**
- One repository interface per entity
- Custom finder methods exist for common queries

**Todo List:**
- [ ] Create `UserRepository` extending `JpaRepository<User, Long>`
- [ ] Create `ItemRepository` extending `JpaRepository<Item, Long>`
- [ ] Create `ItemRequestRepository` — add `findByRequestedBy(User user)` and `findByStatus(RequestStatus status)`
- [ ] Create `NotificationLogRepository`

**Relevant Context:**
- All repositories go in `src/main/java/.../repository/`

**Status:** [ ] pending

---

## Sub-Task 3 — Notification Service

**Intent:** Encapsulate all outbound notification logic behind a single service so the rest of the system never needs to know the channel.

**Expected Outcomes:**
- `NotificationService` inspects the user's `notificationPreference` and delegates to email or SMS
- A `NotificationLog` record is persisted after every notification attempt
- Email sending uses Spring's `JavaMailSender`; SMS sending uses Twilio (or a stub for now)

**Todo List:**
- [ ] Create `NotificationService` with `sendNotification(User user, ItemRequest request)`
- [ ] Implement `sendEmail(User user, String message)` using `JavaMailSender`
- [ ] Implement `sendSMS(User user, String message)` — stub with a log statement initially
- [ ] Persist a `NotificationLog` entry after each send
- [ ] Add mail configuration properties to `application.properties` (host, port, username, password)

**Relevant Context:**
- Notification is triggered only when `ItemRequest.status` transitions to `READY_FOR_PICKUP`
- Log channel as `EMAIL` or `SMS` matching the `NotificationPreference` enum

**Status:** [ ] pending

---

## Sub-Task 4 — Inventory Service

**Intent:** Centralise all business logic for stock checking, request creation, item distribution, and location updates.

**Expected Outcomes:**
- `InventoryService` enforces that a request can only be created when `item.stockAmount > 0`
- `stockAmount` is **decremented immediately** when `submitRequest` succeeds — stock is reserved at that point
- Distributing an item only updates the request status to `DISTRIBUTED`; stock was already decremented at request time
- Marking a request `READY_FOR_PICKUP` triggers the `NotificationService`
- Only admins may update location or distribute

**Todo List:**
- [ ] Create `InventoryService` with the following methods:
  - `checkStock(Long itemId) boolean`
  - `submitRequest(Long userId, Long itemId) ItemRequest` — validates stock > 0, **decrements stockAmount**, creates request with status `PENDING`
  - `acknowledgeRequest(Long requestId) ItemRequest` — admin sets status to `ACKNOWLEDGED`
  - `markReadyForPickup(Long requestId) ItemRequest` — admin sets status, triggers notification
  - `distributeItem(Long requestId) ItemRequest` — admin sets status to `DISTRIBUTED` only (stock already decremented)
  - `updateItemLocation(Long itemId, String newLocation) Item` — admin only
- [ ] Wrap `submitRequest` stock check + decrement in a `@Transactional` with pessimistic locking to prevent race conditions between concurrent requests
- [ ] Throw a descriptive exception when stock is zero and a request is attempted
- [ ] Throw `AccessDeniedException` if a non-admin calls an admin-only method

**Relevant Context:**
- `NotificationService.sendNotification()` is called inside `markReadyForPickup`
- Role check can be done via Spring Security `@PreAuthorize("hasRole('ADMIN')")`

**Status:** [ ] pending

---

## Sub-Task 5 — REST Controllers

**Intent:** Expose the system's capabilities through a clean REST API with appropriate HTTP verbs and status codes.

**Expected Outcomes:**
- `ItemController` exposes item read and admin location-update endpoints
- `RequestController` exposes request creation, status check, and admin lifecycle endpoints
- All endpoints return proper HTTP status codes and JSON responses

**Todo List:**
- [ ] Create `ItemController`:
  - `GET /items` — list all items
  - `GET /items/{id}` — get single item
  - `PATCH /items/{id}/location` — admin updates location
- [ ] Create `RequestController`:
  - `POST /requests` — normal user submits a request (body: userId, itemId)
  - `GET /requests/{id}` — user checks their request status
  - `PATCH /requests/{id}/acknowledge` — admin acknowledges
  - `PATCH /requests/{id}/ready` — admin marks ready for pickup, triggers notification
  - `PATCH /requests/{id}/distribute` — admin distributes
- [ ] Return `409 Conflict` when stock is zero
- [ ] Return `403 Forbidden` for role violations

**Relevant Context:**
- Controllers delegate entirely to `InventoryService`
- Use `ResponseEntity<?>` return types for flexibility

**Status:** [ ] pending

---

## Sub-Task 6 — Security Configuration

**Intent:** Protect admin-only endpoints and ensure users can only access their own data.

**Expected Outcomes:**
- Spring Security is configured with role-based access
- Admin endpoints reject non-admin callers with 403
- Authentication uses HTTP Basic or JWT (Basic is acceptable for initial version)
- Spring Boot 3.x uses `SecurityFilterChain` bean — `WebSecurityConfigurerAdapter` is removed

**Todo List:**
- [ ] Add Spring Security dependency to `pom.xml`
- [ ] Create `SecurityConfig` as a `@Configuration` class with a `SecurityFilterChain` bean (Spring Boot 3.x pattern — no `WebSecurityConfigurerAdapter`)
- [ ] Configure: `PATCH /items/*/location`, `PATCH /requests/*/acknowledge`, `PATCH /requests/*/ready`, `PATCH /requests/*/distribute` — require `ADMIN` role
- [ ] All other endpoints require authentication
- [ ] Wire users from `UserRepository` into a `UserDetailsService`

**Status:** [ ] pending

---

## Sub-Task 7 — Database Migration

**Intent:** Version-control the schema using Flyway or Liquibase so the database is created consistently across environments.

**Expected Outcomes:**
- A migration script creates all four tables with correct columns and constraints
- The application starts cleanly against a fresh database

**Todo List:**
- [ ] Add Flyway dependency to `pom.xml`
- [ ] Create `V1__create_schema.sql` in `src/main/resources/db/migration/` with DDL for `user`, `item`, `item_request`, `notification_log`
- [ ] Add `spring.flyway.enabled=true` to `application.properties`

**Status:** [ ] pending

---

## Diagrams

See chat for the rendered Mermaid class diagram and ER diagram. They are not duplicated here to keep the plan file concise.

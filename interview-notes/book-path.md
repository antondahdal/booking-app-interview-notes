# Book path — read in this order

One Book, from the gateway to the ticket, then what happens after the commit.
This is the call order. The day notes are the quiz. They are not this order.

Backend only.

## 1. Gateway gate

`gateway/src/main/java/com/eventbooking/gateway/BookRateLimitFilter.java`
`doFilterInternal`

Runs only for `POST /api/events/{id}/bookings`.
`tryConsume` spends one token for the caller.
The key is the JWT subject when the request has a bearer token. Otherwise it is the IP.
`false` sets **429** and returns. `chain.doFilter` does not run. Booking is not called. The seat row is not locked.
A failure talking to the jar sets **503** and also returns.

`gateway/src/main/java/com/eventbooking/gateway/RateLimitConfig.java`
`tokenBucket`

Builds the jar. `REDIS_URL` or a Redis host uses `RedisTokenBucket`. No Redis host uses `InMemoryTokenBucket`.
Defaults are 5 tokens, one token back every 15 seconds.

## 2. Gateway route

`gateway/src/main/java/com/eventbooking/gateway/GatewayRouteConfig.java`
`bookingRoutes`

Matches `POST /api/events/{eventId}/bookings` and `/api/bookings/**`.
Forwards the HTTP request to `booking.service.uri`. In the gateway properties that is `http://localhost:8080`.
This method does not call `BookingController`. `BookingController` runs in the booking process after the request arrives.

## 3. Booking controller

`src/main/java/com/eventbooking/event_booking_platform/controller/BookingController.java`
`book`

`@PostMapping` on `/api/events/{eventId}/bookings`.
On the success path it returns **201** (`HttpStatus.CREATED`).
The body is the `BookingResponseDto` from `bookingService.book(dto, eventId)`.

## 4. Booking service

`src/main/java/com/eventbooking/event_booking_platform/service/BookingServiceImpl.java`
`book`

This method is not `@Transactional`. A transaction would hold a database connection during the Auth and Event HTTP calls. Only `writeBook` opens one.

In order:

1. `authClient.checkIfExist()`
2. `bookingWriter.findByClickId(dto.getClickId())`
3. If a booking is already there, `replay`
4. Otherwise `eventClient.reserveSeats(id, dto.getSeats(), dto.getClickId())`
5. `bookingWriter.writeBook(...)`
6. If that save hits `DataIntegrityViolationException`, another try inserted the ticket first. Look the click id up again and `replay`.

### 4a. Who is the caller

`src/main/java/com/eventbooking/event_booking_platform/client/AuthClient.java`
`checkIfExist`

Does not read the token. Copies the `Authorization` header onto `GET /api/users/me`.

`auth/src/main/java/com/eventbooking/auth/controller/UserController.java`
`getUser`

Calls `authService.checkIfExist()`.

`auth/src/main/java/com/eventbooking/auth/service/AuthServiceImpl.java`
`checkIfExist`

The token was already read by Auth's security filter. This method loads the user by the authenticated email. No user throws `ResourceNotFoundException`.
Returns `UserResponseDto` (id, email, role).

### 4b. Same click id, ticket already saved

`BookingServiceImpl.replay`

Same user, same event, same seat count: return that ticket. Event is not called.
A different user, event, or seat count throws `ClickIdMismatchException`. The handler makes that **409**.

## 5. Ask Event to reserve

`src/main/java/com/eventbooking/event_booking_platform/client/EventClient.java`
`reserveSeats`

Sends `POST /api/events/{id}/seat-reservations` with the seat count and the click id.
A **409** whose body says the click id was already used becomes `ClickIdMismatchException`.
Any other **409** becomes `InsufficientSeatsException`.
Returns `EventResponseDto`.

`event/src/main/java/com/eventbooking/events/controller/EventController.java`
`reserveSeats`

Calls `eventService.reserveSeats(id, seats, clickId)` and returns **200** with that DTO.

## 6. Event locks the row and takes seats

`event/src/main/java/com/eventbooking/events/service/EventServiceImpl.java`
`reserveSeats`

1. `eventRepository.findByIdForUpdate(id)` locks that event row. Missing event throws `ResourceNotFoundException`.
2. `seatHoldRepository.findByClickId(clickId)`.
   - Same event id and same seats: `toResponse(event, existing.getId())`. Seats are not subtracted. This is an `EventResponseDto` for the existing hold, not a ticket.
   - Different event id or different seats: `ClickIdMismatchException`. Either mismatch is enough.
3. `checkSeats` is false: `throw new InsufficientSeatsException`. The handler makes that **409**.
4. Enough seats: `availableSeats` minus the requested seats, then save the event.
5. New `SeatHold`: the event, the seats, `setClickId(clickId)`, status `HELD`, `expiresAt` ten minutes from now.
6. `seatHoldRepository.saveAndFlush`. A unique click-id clash throws `ClickIdMismatchException`.
7. `toResponse` returns the event plus the new hold id.

## 7. Write the ticket

`src/main/java/com/eventbooking/event_booking_platform/service/BookingWriter.java`
`writeBook`

`@Transactional`. One commit. Three rows:

1. `bookingRepository.save` — the `Booking`. Status `CONFIRMED`. Click id is set. This is the ticket.
2. `populateMessageAndSave` — an `OutboxMessage` of type `NOTIFY`, status `PENDING`.
3. `populateToConfirm` — an `OutboxMessage` of type `CONFIRM_HOLD`, status `PENDING`, with the hold id.

The mail is not sent here. `CONFIRM_HOLD` is the seat hold, not the event.

Then `publisher.publishEvent(new BookingCreatedEvent(book.getId()))`.
Returns `BookingResponseDto`. That DTO is what `BookingController.book` puts in the **201**.

## 8. After the commit: the log line

`src/main/java/com/eventbooking/event_booking_platform/events/BookingCreatedListener.java`
`EventIdPublisher`

`@TransactionalEventListener(AFTER_COMMIT)`. Runs only if `writeBook` committed.

1. `bookingNotifier.send` prints `ticket` and the booking id. `BookingNotifier.send` does not send email and does not call Event.
2. Loads the `NOTIFY` row and `setStatus("SENT")`.

The `CONFIRM_HOLD` row stays `PENDING`.

## 9. Poller confirms the hold

`src/main/java/com/eventbooking/event_booking_platform/events/OutboxPoller.java`
`poller`

Every 10 seconds. Loads outbox rows with status `PENDING`.

- `NOTIFY`: `bookingNotifier.send` again.
- `CONFIRM_HOLD`: `eventClient.confirmHold(message.getHoldId())`. The hold id, not the ticket. The ticket is already saved.
- Then that row is set to `SENT`.

If confirm throws `HoldDataExceedTimeException`, the booking is set to `CANCELLED`.

`EventClient.confirmHold` posts `/api/events/holds/{holdId}/confirm` with the internal token.

`event/src/main/java/com/eventbooking/events/controller/EventController.java`
`ConfrimHold`

Calls `eventService.confirmHold(holdId)`.

`EventServiceImpl.confirmHold`

Locks the hold. If it is `HELD`, line 174 sets it to `CONFIRMED`. It does not change `availableSeats`.
If it is already `EXPIRED`, it throws `HoldDataExceedTimeException`.

`confirmHold` runs because the ticket and the `CONFIRM_HOLD` row were one commit. No commit means this call does not happen.

## 10. A hold that nobody confirmed

`event/src/main/java/com/eventbooking/events/events/HoldExpiryJob.java`
`expireEvents`

Every 5 seconds.
Line 27 loads holds that are still `HELD` and whose `expiresAt` is already past. They are not status `EXPIRED` yet.
Each one calls `eventService.expireHold`.

`EventServiceImpl.expireHold`

While the hold is `HELD`: add that hold's seats back onto `availableSeats`, and set the hold to `EXPIRED`.
A hold that `confirmHold` already set to `CONFIRMED` is not in this query, so its seats stay taken.

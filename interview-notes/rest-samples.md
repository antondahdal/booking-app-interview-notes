# REST samples

One dummy request for each call this app exposes.
The phone uses the gateway at `http://localhost:8081`.
Two Event calls are not on the gateway. Booking makes those, on port `8083`.
Register creates an `ATTENDEE`. A venue, a new event, and a title change need a token whose role is `ORGANIZER` or `ADMIN`. This list does not create that user.

Replace `PASTE_TOKEN` with the `token` from login.

## Auth

### Register — 201

```http
POST http://localhost:8081/api/auth/register
Content-Type: application/json

{
  "email": "maria@example.com",
  "password": "secret-pass"
}
```

### Login — 200

```http
POST http://localhost:8081/api/auth/login
Content-Type: application/json

{
  "email": "maria@example.com",
  "password": "secret-pass"
}
```

### Current user — 200

```http
GET http://localhost:8081/api/users/me
Authorization: Bearer PASTE_TOKEN
```

## Venues

### Create a hall — 201

Needs an organizer token.

```http
POST http://localhost:8081/api/venues
Authorization: Bearer PASTE_TOKEN
Content-Type: application/json

{
  "name": "Main Hall",
  "address": "1 Market Street",
  "capacity": 500
}
```

### One hall — 200

```http
GET http://localhost:8081/api/venues/1
```

### All halls — 200

```http
GET http://localhost:8081/api/venues
```

## Events

### Create a concert — 201

Needs an organizer token. `dateTime` is local, no timezone suffix.

```http
POST http://localhost:8081/api/events
Authorization: Bearer PASTE_TOKEN
Content-Type: application/json

{
  "venueId": 1,
  "title": "Night Show",
  "dateTime": "2026-11-01T20:00:00",
  "totalSeats": 100
}
```

### One concert — 200

```http
GET http://localhost:8081/api/events/1
```

### A page of concerts — 200

```http
GET http://localhost:8081/api/events?page=0&size=20
```

### Change the title — 200

Needs an organizer token. The body is only the title.

```http
PATCH http://localhost:8081/api/events/1
Authorization: Bearer PASTE_TOKEN
Content-Type: application/json

{
  "title": "Night Show Late"
}
```

## Bookings

### Book — 201

Needs an attendee token. `seats` must be at least 1. The same `clickId` is the same tap.

```http
POST http://localhost:8081/api/events/1/bookings
Authorization: Bearer PASTE_TOKEN
Content-Type: application/json

{
  "seats": 2,
  "clickId": "tap-maria-001"
}
```

### My tickets — 200

```http
GET http://localhost:8081/api/bookings/me
Authorization: Bearer PASTE_TOKEN
```

## Gateway

### Health — 200

No token. The body is the text `ok`.

```http
GET http://localhost:8081/health
```

## Event, called by Booking only

These two are not routed on the gateway. A phone that calls them directly is not the normal path.

### Reserve seats — 200

```http
POST http://localhost:8083/api/events/1/seat-reservations
Authorization: Bearer PASTE_TOKEN
Content-Type: application/json

{
  "seats": 2,
  "clickId": "tap-maria-001"
}
```

### Confirm a hold — 200

The token role is `INTERNAL`. No body. The reply body is the text `Ok`.

```http
POST http://localhost:8083/api/events/holds/10/confirm
Authorization: Bearer PASTE_TOKEN
```

# Vehicle Rental Microservices

A Python Flask backend demonstrating a vehicle rental workflow using independently running services, an API gateway, service discovery, and separate SQLite databases.

The project explores how bookings coordinate with user, vehicle, and payment services, including failure handling through a circuit breaker and Saga-style compensation.

## Features

- User creation, retrieval, update, and deletion.
- Vehicle management with availability, reservation, and release operations.
- Booking creation with user and vehicle verification.
- Mock payment processing with success and failure scenarios.
- API gateway routing through a service registry.
- Circuit breaker for payment-service communication failures.
- Compensation that attempts to release reserved vehicles when payment fails.
- Versioned vehicle APIs.
- Postman collection for manual API testing.

## Architecture

```text
                        Client / Postman
                               |
                               v
                       API Gateway :5000
                               |
                   Service discovery lookup
                               |
                      Service Registry :5005

          Gateway forwards requests to the discovered service
                               |
             +-----------------+------------------+
             |                 |                  |
             v                 v                  v
      User Service      Vehicle Service    Booking Service
         :5001              :5002              :5003
             |                 |                  |
          user.db          vehicle.db         booking.db
                                                  |
                                                  v
                                          Payment Service
                                               :5004
                                                  |
                                              payment.db
```

The Booking Service also discovers and calls the User, Vehicle, and Payment services to coordinate a rental.

## Technology Stack

| Area | Technology |
|---|---|
| Language | Python |
| Web framework | Flask |
| HTTP communication | Requests |
| Persistence | SQLite |
| API testing | Postman |
| Version control | Git and GitHub |

## Project Structure

```text
vehicle-rental-microservices/
├── api-gateway/
│   └── app.py
├── service-registry/
│   └── app.py
├── user-service/
│   └── app.py
├── vehicle-service/
│   └── app.py
├── booking-service/
│   ├── app.py
│   └── saga.py
├── payment-service/
│   ├── app.py
│   └── circuit_breaker.py
├── postman/
│   └── vehicle-rental-api.json
├── docs/
├── tests/
│   └── test_api.py
├── requirements.txt
└── README.md
```

## Installation

### Clone the repository

```bash
git clone https://github.com/itzsonu/vehicle-rental-microservices.git
cd vehicle-rental-microservices
```

### Install dependencies

```bash
python -m pip install flask requests
```

The current `requirements.txt` is empty, so install the packages directly using the command above.

SQLite support is included with Python.

## Running the Services

Open **six separate terminals**, all in the repository root.

Start the Service Registry first, then the remaining services in the order shown below. Leave each process running.

| Terminal | Service | Command | Port |
|---|---|---|---:|
| 1 | Service Registry | `python service-registry/app.py` | 5005 |
| 2 | User Service | `python user-service/app.py` | 5001 |
| 3 | Vehicle Service | `python vehicle-service/app.py` | 5002 |
| 4 | Payment Service | `python payment-service/app.py` | 5004 |
| 5 | Booking Service | `python booking-service/app.py` | 5003 |
| 6 | API Gateway | `python api-gateway/app.py` | 5000 |

Ensure ports 5000–5005 are available before starting.

The services use relative SQLite database paths. Run these commands consistently from the repository root to use the database files in that directory.

Services register themselves during startup. If the registry is restarted, restart the four business services so they register again.

## Verify Startup

Check the gateway:

```text
GET http://localhost:5000/health
```

Inspect registered services:

```text
GET http://localhost:5005/services
```

Each business service also exposes `/health` on its own port.

## Main API Endpoints

Send application requests through the gateway at:

```text
http://localhost:5000
```

| Method | Endpoint | Purpose |
|---|---|---|
| GET, POST | `/api/v1/users` | List or create users |
| GET, PUT, DELETE | `/api/v1/users/{id}` | Retrieve, update, or delete a user |
| GET, POST | `/api/v1/vehicles` | List or create vehicles |
| GET, PUT, DELETE | `/api/v1/vehicles/{id}` | Retrieve, update, or delete a vehicle |
| PUT | `/api/v1/vehicles/{id}/reserve` | Reserve a vehicle |
| PUT | `/api/v1/vehicles/{id}/release` | Release a vehicle |
| GET, POST | `/api/v1/bookings` | List or create bookings |
| GET | `/api/v1/bookings/{id}` | Retrieve a booking |
| GET | `/api/v1/bookings/circuit-breaker` | Inspect payment circuit breaker state |
| GET, POST | `/api/v1/payments` | List payments or process a mock payment |
| GET | `/api/v1/payments/{id}` | Retrieve a payment |
| GET | `/api/v2/vehicles` | List vehicles using V2 |
| GET | `/api/v2/vehicles/{id}` | Retrieve a vehicle using V2 |
| GET | `/api/v2/vehicles/{id}/summary` | Retrieve a vehicle summary |

V2 routing is currently available only for Vehicle Service endpoints.

## Example Booking Workflow

Use Postman with **Body → raw → JSON** for POST requests.

### 1. Create a user

```text
POST http://localhost:5000/api/v1/users
```

```json
{
  "name": "Demo User",
  "email": "rental-demo@example.com",
  "phone": "9000000000"
}
```

Save the returned user ID. Use a different email if this one already exists.

### 2. Create a vehicle

```text
POST http://localhost:5000/api/v1/vehicles
```

```json
{
  "name": "Demo Sedan",
  "vehicle_number": "DEMO-RENTAL-001",
  "vehicle_type": "CAR",
  "price_per_day": 1500,
  "status": "AVAILABLE"
}
```

Save the returned vehicle ID. Vehicle numbers must be unique.

### 3. Create a booking

```text
POST http://localhost:5000/api/v1/bookings
```

Replace the IDs below with those returned by the previous requests.

```json
{
  "user_id": 1,
  "vehicle_id": 1,
  "start_date": "2026-11-01",
  "end_date": "2026-11-02",
  "payment_status": "SUCCESS"
}
```

Expected behavior:

1. Verify the user.
2. Verify that the vehicle is available.
3. Reserve the vehicle.
4. Create a pending booking.
5. Process the mock payment.
6. Confirm the booking when payment succeeds.

A successful booking returns HTTP 201.

## Failure Handling

### Mock payment failure

Create a booking for an available vehicle with:

```json
"payment_status": "FAILED"
```

Expected behavior:

- The mock payment fails.
- The booking becomes `CANCELLED`.
- Compensation attempts to release the vehicle.
- The booking request returns HTTP 402.

Use a new available vehicle or release the earlier vehicle before repeating a booking test.

### Payment Service outage

Stop the Payment Service while leaving the other services running. Submit otherwise valid booking requests.

Payment communication failures are recorded by the circuit breaker. Its configured failure threshold is **3**, with a **10-second recovery timeout**.

Inspect its state through:

```text
GET http://localhost:5000/api/v1/bookings/circuit-breaker
```

The circuit uses `CLOSED`, `OPEN`, and `HALF-OPEN` states. Recovery is evaluated when another booking request reaches the circuit breaker.

A mock payment rejection is handled separately from a communication failure.

## Postman Testing

Import this collection into Postman:

```text
postman/vehicle-rental-api.json
```

Start all services before running requests. Check the collection's URLs and replace sample IDs with records that exist in your local databases.

The current `tests/test_api.py` is empty; automated test coverage is not included.

## Current Scope

- Payments are simulated; no real payment provider is connected.
- Booking pricing currently uses one `price_per_day` value rather than calculating the full rental duration.
- The registry and circuit breaker keep their state in memory.
- Compensation is an attempted recovery action and can itself fail if Vehicle Service is unavailable.
- The application currently provides backend APIs without a frontend.
- Flask debug servers are configured for local development.

## Planned Improvements

- Add automated integration tests.
- Calculate rental charges from validated booking dates.
- Add authentication and authorization.
- Move configuration into environment variables.
- Add durable recovery for failed compensation.
- Containerize services and provide a repeatable startup workflow.

## Attribution

This repository is a learning copy of the original project by **mnmomin123**, maintained here for study and further development.

- [Original repository](https://github.com/mnmomin123/vehicle-rental-microservices)
- [This repository](https://github.com/itzsonu/vehicle-rental-microservices)

Original commit history and author credits are retained.

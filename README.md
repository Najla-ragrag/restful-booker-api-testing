# restful-booker-api-testing
# Restful-Booker API — Automated Test Suite with CI/CD

![API Tests](https://github.com/Najla-ragrag/restful-booker-api-testing/actions/workflows/api-tests.yml/badge.svg)

An API test suite for the public [Restful-Booker](https://restful-booker.herokuapp.com/apidoc/index.html) booking API, built with **Postman + Newman + GitHub Actions**. It covers the full booking lifecycle (authenticate, create, read, update, partially update, delete, verify deletion) and a set of negative tests, and runs automatically on every push.

## Results

| Run | Requests | Assertions | Failed |
|-----|----------|------------|--------|
| Full collection (Newman, GitHub Actions) | 14 | 30 | 0 |

An HTML report is uploaded as a build artifact on every run.

## Tech Stack

- **Testing:** Postman (collections, environments, pre-request and post-response scripts, Collection Runner)
- **Automation/CLI:** Newman + newman-reporter-htmlextra
- **CI/CD:** GitHub Actions (Ubuntu runner, Node.js 20)
- **API under test:** `restful-booker.herokuapp.com`
- **Auth:** token sent in the `Cookie` header (`token=<value>`)

## Test Coverage

### Booking lifecycle (9 requests, 19 assertions)

| Request | Method | Endpoint | Expected result |
|---------|--------|----------|-----------------|
| Ping | GET | `/ping` | 201 |
| Get All Booking Numbers | GET | `/booking` | 200, list of booking ids |
| Create Token | POST | `/auth` | 200, non-empty token (saved for later requests) |
| Create Booking | POST | `/booking` | 200, response matches the request (booking id saved) |
| Get Booking Detail | GET | `/booking/{id}` | 200, belongs to the booking just created |
| Update Booking | PUT | `/booking/{id}` | 200, all fields updated |
| Partial Update Booking | PATCH | `/booking/{id}` | 200, price changed, other fields kept |
| Delete Booking | DELETE | `/booking/{id}` | 201, `Created` |
| Verify Booking Deleted | GET | `/booking/{id}` | 404, `Not Found` |

### Negative tests (5 requests, 11 assertions)

| Scenario | Observed result |
|----------|-----------------|
| DELETE without a token (non-existent booking) | 403 `Forbidden` |
| PUT with an invalid token | 403 `Forbidden` |
| GET a booking that does not exist | 404 `Not Found` |
| Create token with a wrong password | 200 with `{"reason": "Bad credentials"}` and no token |
| Create a booking with required fields missing | 500 `Internal Server Error` |

The negative tests do not depend on a token or a booking created earlier in the run, so they can run in any order.

## Findings

### Likely bug
- **Missing required fields return 500.** Creating a booking without `firstname` and `lastname` returns `500 Internal Server Error`. A client error like this should return a 4xx. The test asserts the current behaviour and is marked as a known issue.

### API behaviour notes
- `GET /ping` returns **201 Created** and a successful delete returns **201 Created**. Both match the API documentation. The documentation's own tables are labelled "Success 200" while the example responses show 201.
- Creating a booking returns **200** rather than the more conventional 201.
- Unauthorised requests return **403 Forbidden**, not 401, and the authorisation check happens **before** the booking lookup (a DELETE without a token on a non-existent booking returns 403, not 404).
- 403 and 404 responses are plain text (`Forbidden`, `Not Found`), not JSON.

### Undocumented behaviour
- A **wrong password returns 200** with `{"reason": "Bad credentials"}` instead of 401. The documentation for the token endpoint only describes the success response.

### Open question
- In two manual Postman Collection Runner runs, `DELETE` returned 403 even with a fresh token, then succeeded in later runs. The cause was not identified. It has not appeared in CI so far.

## Automation Highlights

**Token capture** (Create Token, post-response):
```javascript
const res = pm.response.json();
pm.environment.set("token", res.token);
```

**Booking id capture** (Create Booking, post-response):
```javascript
pm.environment.set("bookingid", res.bookingid);
```

**Unique test data** (pre-request):
```javascript
pm.environment.set("firstname", "Tester" + Date.now());
```

**Asserting against what was sent** (Create Booking, post-response):
```javascript
const sent = JSON.parse(pm.variables.replaceIn(pm.request.body.raw));
pm.test("Booking data matches the request", function () {
    pm.expect(res.booking.firstname).to.eql(sent.firstname);
});
```

## CI/CD Pipeline

`.github/workflows/api-tests.yml` runs on every push to `main` (or manually from the Actions tab):

1. Checks out the repo and sets up Node.js 20
2. Installs Newman and the HTML reporter
3. Runs the collection against the environment file
4. Uploads the HTML report as an artifact

The public API resets its data every few minutes, so every run creates its own token and booking instead of relying on existing data.

## How to Run

**In Postman:** import `restful-booker.postman_collection.json` and `restful-booker.postman_environment.json`, select the environment, and run the collection with the Collection Runner.

**With Newman:**
```bash
npm install -g newman newman-reporter-htmlextra
newman run "restful-booker.postman_collection.json" \
  -e "restful-booker.postman_environment.json" \
  -r cli,htmlextra --reporter-htmlextra-export report.html
```

## Skills Demonstrated

- REST API testing: GET, POST, PUT, PATCH, DELETE
- Positive and negative test design, including authorisation and invalid input
- Token-based authentication flow and request chaining with environment variables
- Pre-request and post-response scripting in JavaScript
- Test independence and test-data management on a shared public API
- Reading API documentation and separating bugs, behaviour notes and undocumented behaviour
- CLI automation with Newman and CI/CD with GitHub Actions

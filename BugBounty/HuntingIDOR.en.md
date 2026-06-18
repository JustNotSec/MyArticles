---
title: "Hunting IDOR: When the Server Trusts the Client Too Much"
dek: "IDOR is the bug that survives every framework upgrade. Here is how I find it, why it hides in plain sight, and the one question that turns a 200 OK into a finding."
date: 2026-06-18
tags: [BugBounty, IDOR, Access Control, Web]
venue: BugBounty
reading_time: 9 min
draft: false
---

Insecure Direct Object Reference is the bug everyone has read about and almost
everyone still ships. It is not exotic. There is no clever payload, no encoding
trick, no race window to thread. You change a number, and the server hands you
data that was never yours. The reason it persists is not technical difficulty.
It is that authorization is the one control a framework cannot put on
autopilot for you.

This is how I approach IDOR on a real target, from the first request to a
report that survives triage.

## What IDOR actually is

Strip away the acronym and it is one sentence: **the application uses a
client-supplied identifier to fetch an object, and forgets to check whether the
current user is allowed to have it.**

```http
GET /api/invoices/10423 HTTP/1.1
Authorization: Bearer <alice-token>
```

The server reads `10423`, looks it up, and returns it. It authenticated Alice —
it knows *who* she is. It just never asked whether invoice `10423` *belongs* to
her. Authentication answered "who are you." Authorization should have answered
"are you allowed to see this," and nobody asked.

That gap is the whole bug.

## Why frameworks do not save you

Modern stacks give you authentication almost for free. A decorator, a
middleware, a session guard — drop it in and every route demands a valid token.
That is the trap. Teams see the green padlock on every endpoint and assume
access control is handled.

But authentication is global and mechanical. Authorization is per-object and
contextual. No framework can know that invoice `10423` belongs to org `88` and
that Alice is only a member of org `42`. That logic lives in *your* domain, so
*you* have to write the check — on every single object lookup. Miss one, and
that endpoint is an IDOR.

> The bug is not that the developer wrote bad code. It is that they wrote
> *no* code where a check was required, and nothing in the toolchain
> complained.

## Finding it: map, then mutate

My workflow is boring on purpose. Boring is repeatable.

### 1. Inventory every identifier

Browse the app as a normal user with a proxy running. Catalog every request
that carries an object reference:

- Numeric path params: `/orders/5512`, `/users/77/settings`
- UUIDs: `/files/3f9a-...` (yes, these are guessable far more often than people think)
- IDs in query strings: `?account_id=42`
- IDs hidden in JSON bodies: `{"projectId": 19}`
- IDs in headers: `X-Account-Context: 42`

The ones people forget are bodies and headers. Hunters who only fuzz the URL
miss half the surface.

### 2. Use two accounts, always

This is the single most important habit. Register **two** independent accounts,
ideally in two separate orgs/tenants. Call them the *attacker* (Alice) and the
*victim* (Bob).

Capture a request as Bob that returns one of his objects:

```http
GET /api/invoices/20001 HTTP/1.1
Authorization: Bearer <bob-token>
```

Now replay it with **Alice's token** but **Bob's object id**:

```http
GET /api/invoices/20001 HTTP/1.1
Authorization: Bearer <alice-token>
```

### 3. Ask the one question

When that second request comes back, ask:

**Did Alice just receive Bob's data?**

- `200 OK` with Bob's invoice body → confirmed IDOR.
- `403 / 404` → the check exists, move on.
- `200 OK` with an empty or filtered body → partial; probe further.

That is the entire test. Authentication is intact the whole time — Alice is
fully logged in. You are testing the layer above it.

## The variations that pay

Once the basic read works, the same flaw usually rots through the rest of the
object's lifecycle. Test every verb, not just `GET`:

| Method   | What you are testing                              |
| -------- | ------------------------------------------------- |
| `GET`    | Read someone else's object                        |
| `PUT`    | Overwrite someone else's object                   |
| `DELETE` | Destroy someone else's object                     |
| `POST`   | Create an object *inside* someone else's resource |

A read-only IDOR is a leak. A `DELETE` IDOR is an outage. Triage teams pay for
the difference, so always demonstrate the highest-impact verb the endpoint
accepts.

A few more places it hides:

- **Mass assignment cousins.** `PATCH /users/me` with a body of
  `{"org_id": 42}` — can you move yourself into another org?
- **Nested references.** `/orgs/42/projects/19` may check org `42` but trust
  `19` blindly. Swap only the inner id.
- **Predictable "random" ids.** Sequential UUIDv1, timestamp-based ids, or
  base64 of an integer are not unguessable. Decode before you assume safety.
- **Export and report endpoints.** PDF/CSV generators are notorious for
  skipping the ownership check the main view enforces.

## Writing the report

A confirmed IDOR with a weak report gets downgraded. Make triage trivial:

1. **State the two identities.** "Account A (attacker) and Account B (victim),
   no relationship between them."
2. **Show the request.** Alice's token, Bob's object id, full headers.
3. **Show the response.** Bob's data, with a field that proves ownership
   (his email, his org name).
4. **State the impact in their language.** "Any authenticated user can read,
   modify, or delete any invoice belonging to any other tenant by incrementing
   the `id` path parameter."

That last sentence is what moves severity. You are not reporting a `200 OK`.
You are reporting cross-tenant data access with no authorization boundary.

## The fix, so you can speak to it

When the program asks how to remediate — and good ones do — the answer is
always the same shape: **scope every object lookup to the current principal.**

```sql
-- vulnerable: trusts the id alone
SELECT * FROM invoices WHERE id = :id;

-- fixed: the id AND the owner must match
SELECT * FROM invoices WHERE id = :id AND org_id = :current_org;
```

Do the check at the data layer, not just the controller, so it cannot be
bypassed by a route that forgot its guard. Centralize it. A per-query ownership
filter that the ORM applies automatically beats a hand-written `if` on every
endpoint, because the bug is precisely the endpoint that forgot the `if`.

---

IDOR rewards patience over cleverness. Two accounts, a proxy, and the
discipline to swap one identifier at a time will find more real bugs than any
automated scanner. The server trusts the client too much. Your whole job is to
prove it.

# Website leads API

`POST /api/website/save-event-enquiry/` creates one new lead for the company that owns the API key.

The caller sends the same fields for every company. The API reads that company's `business_type` and writes the lead to the matching model:

- `events` (the default) creates an `Enquiry` and, when `slug` matches, links that company's listing on `event`. The listing name is copied to `eventname`.
- `stays` creates a `PropertyEnquiry` and, when `slug` matches, links that company's listing on `property`. The name shown in the CRM is the linked listing's name. `source` is `website`.

The call always inserts a new lead. It does not update an existing one. A customer profile with the same email or mobile on that company is reused and updated.

`POST /tours/save-event-enquiry/<slug>/` is a different Logout page form. The slug there is already in the URL, and that form does not use this API key.

## Endpoint

```
POST https://logout.world/api/website/save-event-enquiry/
```

Accepted bodies:

- `application/json`
- `application/x-www-form-urlencoded`
- `multipart/form-data`

Any other method returns **405**:

```json
{"detail": "Method \"GET\" not allowed."}
```

## Authentication

The company is the `CompanyDetails` row whose `api_key` matches. Send the key in one of these places. The first one that is present is used:

1. Header `X-Api-Key`
2. Body field `api_key`
3. Query parameter `api_key`

No key:

```json
{"detail": "Please provide api_key"}
```

HTTP **403**.

A key that matches no company:

```json
{"detail": "Not found."}
```

HTTP **404**.

## Request fields

None of the body fields are required. Omitted fields are stored empty, and the lead is still created.

| Field | Where | Stored as |
| --- | --- | --- |
| `fullname` | Body | Customer profile. An existing profile keeps its current name when this is omitted. |
| `email` | Body | Lowercased, then saved on the profile. Used to find an existing profile. |
| `mobile` | Body | Customer profile. Used to find an existing profile. The WhatsApp alert is skipped when this is empty. |
| `message` | Body | Lead message. When this is omitted or blank, `is_pdf_downloaded` is stored as `true`. |
| `is_expecting_call_back` | Body | See the boolean rules below. Also gates the WhatsApp alert. |
| `no_of_guests` | Body | Text on the lead. A JSON number is accepted. |
| `preferred_start_date` | Body | On an events company, `Enquiry.preferred_start_date`. On a stays company, `PropertyEnquiry.checkin_date`. |
| `from_url` | Body | `origin_domain`. When omitted, the `Referer` header is stored instead. |
| `slug` | Body or query string | The listing to attach. The company type decides which model receives it. See below. |
| `api_key` | Header, body, or query | Identifies the company. See Authentication. |

An id in the body is ignored. The listing is attached only from `slug`.

### `slug`

Send the slug stored on that company's listing, including any suffix. `triund-trek-ztod` matches a listing whose slug is `triund-trek-ztod`. `triund-trek` does not match that listing.

The lookup stays inside the company identified by the API key. After the company type is known:

- An events company looks for an `EventsDetails` row with that slug and sets `Enquiry.event` and `Enquiry.eventname`.
- A stays company looks for a `Property` row with that slug and sets `PropertyEnquiry.property`.

A slug that matches nothing still returns **200** and still creates the lead. The listing link is left empty.

### `is_expecting_call_back`

| Sent value | Stored |
| --- | --- |
| JSON `true`, or the string `1`, `true`, `yes`, `on` (any letter case) | `true` |
| Omitted, JSON `false`, or any other string, including `false` | `false` |

### `preferred_start_date`

Send `YYYY-MM-DD`, for example `2026-10-20`. The value is parsed with `dateutil`. A value that cannot be parsed fails the request with HTTP **500** before the lead or the customer profile is written.

### Customer profile

Matching is inside this company, and deleted profiles are skipped.

- Both `email` and `mobile` are sent: the first profile whose email or mobile matches.
- Only one of them is sent: match on that field.
- Neither matches: a new profile is created for this company.

`fullname`, `email`, and `mobile` on the matched profile are overwritten when the request includes them.

## Where the lead is stored

Shared fields on either model: `companyname` from the API key, `customer_profile`, `message`, `no_of_guests`, `is_expecting_call_back`, `origin_domain`, and `is_pdf_downloaded` (`true` when `message` is empty).

### Events company

A new `Enquiry`. `preferred_start_date` is stored on that field. A matching `slug` sets `event` and copies the listing name into `eventname`.

A sales rep is then assigned:

1. The matched listing's assignment pool, when it has available reps. A rep already on an earlier lead for this customer is kept when that rep is available and is in the pool.
2. Otherwise a rep from an earlier lead for this customer, when that rep is available on this company.
3. Otherwise the company's default round-robin pool, when round robin is turned on and that pool has available reps.

If none of those produce a rep, the lead is saved with no assignee.

### Stays company

A new `PropertyEnquiry` with `source` `website`. `preferred_start_date` is stored as `checkin_date`. A matching `slug` sets `property`.

A sales rep is then assigned:

1. The rep on this customer's most recent other stays lead, when that rep is available and is on this company's sales team.
2. Otherwise the matched listing's assignment pool.
3. Otherwise the company's default pool, when round robin is turned on.

If none of those produce a rep, the lead is saved with no assignee.

## Notifications

After the lead is saved, a WhatsApp alert to the sales rep is queued. The HTTP response does not wait for that send, and a failed send does not change the **200**.

The alert is skipped when:

- the customer profile has no `mobile`, or
- `is_expecting_call_back` is `false` and the company does not have the `leadnotifications` add-on enabled.

On an events company the alert goes to the assigned rep's phone, or to the company's WhatsApp number when the rep has no phone. On a stays company it goes to the assigned rep's phone only.

## Response

Success is HTTP **200** for both company types:

```json
{"message": "Success"}
```

The body does not include the new lead id or the customer profile id.

| Situation | Status | Body |
| --- | --- | --- |
| Lead created, including a slug that matched nothing | 200 | `{"message": "Success"}` |
| API key missing | 403 | `{"detail": "Please provide api_key"}` |
| API key matches no company | 404 | `{"detail": "Not found."}` |
| Method is not POST | 405 | `{"detail": "Method \"GET\" not allowed."}` |
| `preferred_start_date` cannot be parsed | 500 | The lead is not created. |

## Example

The request is the same for an events company and a stays company. The API key decides which model is written.

```bash
curl -X POST 'https://logout.world/api/website/save-event-enquiry/?api_key=YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "fullname": "Ada Lovelace",
    "email": "ada@example.com",
    "mobile": "919876543210",
    "slug": "triund-trek-ztod",
    "no_of_guests": 2,
    "preferred_start_date": "2026-10-20",
    "message": "Interested in this listing",
    "is_expecting_call_back": true,
    "from_url": "https://example.com/listing/triund-trek-ztod"
  }'
```

`slug` can also be a query parameter, which is the usual place for a form-encoded body:

```bash
curl -X POST 'https://logout.world/api/website/save-event-enquiry/?api_key=YOUR_API_KEY&slug=triund-trek-ztod' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'fullname=Ada Lovelace' \
  -d 'email=ada@example.com' \
  -d 'mobile=919876543210' \
  -d 'no_of_guests=2' \
  -d 'preferred_start_date=2026-10-20' \
  -d 'message=Interested in this listing' \
  -d 'is_expecting_call_back=true' \
  -d 'from_url=https://example.com/listing/triund-trek-ztod'
```

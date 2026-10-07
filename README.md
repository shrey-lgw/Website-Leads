# Website leads API

`POST /api/website/save-event-enquiry/` creates one new lead for the company that owns the API key.

The company record decides the lead type:

- `business_type` `events` (the default) creates an `Enquiry`.
- `business_type` `stays` creates a `PropertyEnquiry`.

The call always inserts a new lead. It does not update an existing one. A customer profile with the same email or mobile on that company is reused and updated.

This is the API-key endpoint. `POST /tours/save-event-enquiry/<event-slug>/` is a different Logout page form. The event slug there is the URL path, and that form does not use this API key.

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

| Field | Where | Events company | Stays company |
| --- | --- | --- | --- |
| `fullname` | Body | Saved on the customer profile. An existing profile keeps its current name when this is omitted. | Same. |
| `email` | Body | Lowercased, then saved on the profile. Used to find an existing profile. | Same. |
| `mobile` | Body | Saved on the profile. Used to find an existing profile. A WhatsApp alert to the sales rep is skipped when this is empty. | Same. |
| `message` | Body | Saved on the lead. When this is omitted or blank, `is_pdf_downloaded` is stored as `true`. | Same. |
| `is_expecting_call_back` | Body | See the boolean rules below. Also gates the WhatsApp alert. | Same. |
| `no_of_guests` | Body | Stored as text. A JSON number is accepted. | Same. |
| `preferred_start_date` | Body | Parsed and stored on `Enquiry.preferred_start_date`. | Parsed and stored on `PropertyEnquiry.checkin_date`. |
| `from_url` | Body | Stored as `origin_domain`. When omitted, the `Referer` header is stored instead. | Same. |
| `event_slug` | Body or query string | Looks up `EventsDetails.slug` for this company. On a match, the lead's `event` is that row and `eventname` is that event's name. | Looks up `Property.slug` for this company. On a match, the lead's `property` is that row. |
| `slug` | Body or query string | Ignored. | Used only when `event_slug` is absent. Same property lookup as `event_slug`. |
| `api_key` | Header, body, or query | Identifies the company. See Authentication. | Same. |

`event_id` is not read. Sending it does not attach an event or a property. Send `event_slug`.

### `event_slug`

The value must be the slug stored on the event or property, including any suffix. `triund-trek-ztod` matches the event whose slug is `triund-trek-ztod`. `triund-trek` does not match that event.

The lookup is limited to the company identified by the API key. A slug that belongs to another company is a miss.

A miss still returns **200** and still creates the lead. The event or property link is left empty.

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

## What gets saved

### Events company

A new `Enquiry` with:

- `companyname` from the API key
- `customer_profile` from the match above
- `event` and `eventname` when `event_slug` matches
- `message`, `no_of_guests`, `preferred_start_date`, `is_expecting_call_back`
- `origin_domain` from `from_url`, otherwise the `Referer` header
- `is_pdf_downloaded` `true` when `message` is empty

A sales rep is then assigned:

1. The event's assignment pool, when the matched event has one and it has available reps. A rep already on an earlier lead for this customer is kept when that rep is available and is in the pool.
2. Otherwise a rep from an earlier lead for this customer, when that rep is available on this company.
3. Otherwise the company's default round-robin pool, when round robin is turned on for the company and that pool has available reps.

If none of those produce a rep, the lead is saved with no assignee.

### Stays company

A new `PropertyEnquiry` with `source` set to `website`, and:

- `companyname` from the API key
- `customer_profile` from the match above
- `property` when `event_slug` or `slug` matches a property of this company
- `message`, `no_of_guests`, `is_expecting_call_back`
- `checkin_date` from `preferred_start_date`
- `origin_domain` from `from_url`, otherwise the `Referer` header
- `is_pdf_downloaded` `true` when `message` is empty

`PropertyEnquiry` has no `eventname` field. The property name shown in the CRM is the linked property's name.

A sales rep is then assigned:

1. The rep on this customer's most recent other property lead, when that rep is available and is on this company's sales team.
2. Otherwise the matched property's assignment pool.
3. Otherwise the company's default property pool, when round robin is turned on.

If none of those produce a rep, the lead is saved with no assignee.

## Notifications

After the lead is saved, a WhatsApp alert to the sales rep is queued. The HTTP response does not wait for that send, and a failed send does not change the **200**.

The alert is skipped when:

- the customer profile has no `mobile`, or
- `is_expecting_call_back` is `false` and the company does not have the `leadnotifications` add-on enabled.

For an events company the alert goes to the assigned rep's phone, or to the company's WhatsApp number when the rep has no phone. For a stays company it goes to the assigned rep's phone only.

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

## Examples

Events company, form body, slug on the query string:

```bash
curl -X POST 'https://logout.world/api/website/save-event-enquiry/?api_key=YOUR_API_KEY&event_slug=triund-trek-ztod' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'fullname=Ada Lovelace' \
  -d 'email=ada@example.com' \
  -d 'mobile=919876543210' \
  -d 'no_of_guests=2' \
  -d 'preferred_start_date=2026-10-20' \
  -d 'message=Interested in Triund' \
  -d 'is_expecting_call_back=true' \
  -d 'from_url=https://example.com/activity/triund-trek-ztod'
```

Events company, JSON body:

```bash
curl -X POST 'https://logout.world/api/website/save-event-enquiry/' \
  -H 'Content-Type: application/json' \
  -H 'X-Api-Key: YOUR_API_KEY' \
  -d '{
    "fullname": "Ada Lovelace",
    "email": "ada@example.com",
    "mobile": "919876543210",
    "event_slug": "triund-trek-ztod",
    "no_of_guests": 2,
    "preferred_start_date": "2026-10-20",
    "message": "Interested in Triund",
    "is_expecting_call_back": true,
    "from_url": "https://example.com/activity/triund-trek-ztod"
  }'
```

Stays company. `event_slug` is the property slug. `slug` is only a fallback when `event_slug` is omitted.

```bash
curl -X POST 'https://logout.world/api/website/save-event-enquiry/?api_key=YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "fullname": "Ada Lovelace",
    "email": "ada@example.com",
    "mobile": "919876543210",
    "event_slug": "cedar-cottage",
    "no_of_guests": 2,
    "preferred_start_date": "2026-10-20",
    "message": "Two nights",
    "is_expecting_call_back": true,
    "from_url": "https://example.com/stay/cedar-cottage"
  }'
```

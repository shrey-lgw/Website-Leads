
# Capture Webiste Leads API Documentation

## Save Event Enquiry
Creates or updates an event enquiry from website forms and handles customer profile creation/updates.

### Endpoint
```
POST https://logout.world/website/save-event-enquiry/
```

### Authentication
- Required
- Type: API Key
- Location: Query Parameter (`api_key`)

### Request Parameters

#### Query Parameters
| Parameter | Type   | Required | Description           |
|-----------|--------|----------|-----------------------|
| api_key   | string | Yes      | Company's unique API key |

#### Request Body
| Field                 | Type    | Required | Description                                      |
|----------------------|---------|----------|--------------------------------------------------|
| fullname             | string  | Yes      | Customer's full name                             |
| email                | string  | Yes*     | Customer's email address (*required if no mobile) |
| mobile               | string  | Yes*     | Customer's mobile number (*required if no email)  |
| message              | string  | No       | Enquiry message/details                          |
| event_id             | integer | No       | ID of the specific event                         |
| is_expecting_call_back| boolean| No       | Whether customer expects a callback              |
| no_of_guests         | integer | No       | Number of guests/participants                    |
| preferred_start_date | string  | No       | Preferred start date (format: YYYY-MM-DD)        |
| from_url             | string  | No       | Source URL of the enquiry                        |

### Response

#### Success Response (201 Created)
```json
{
    "status": "success",
    "message": "Enquiry saved successfully",
    "data": {
        "enquiry_id": 123,
        "customer_profile_id": 456
    }
}
```

#### Error Responses

##### Invalid Method (405)
```json
{
    "status": "error",
    "message": "Method not allowed"
}
```

##### Authentication Error (401)
```json
{
    "status": "error",
    "message": "API key is required"
}
```

##### Validation Error (400)
```json
{
    "status": "error",
    "message": "Missing required fields: fullname, contact"
}
```

##### Invalid Date Format (400)
```json
{
    "status": "error",
    "message": "Invalid date format for preferred_start_date"
}
```

##### Server Error (500)
```json
{
    "status": "error",
    "message": "An error occurred while processing your request"
}
```

### Example Request

```curl
curl -X POST 'https://your-domain.com/api/save-event-enquiry/?api_key=your-api-key' \
-H 'Content-Type: application/x-www-form-urlencoded' \
-d 'fullname=John Doe' \
-d 'email=john@example.com' \
-d 'mobile=1234567890' \
-d 'event_id=123' \
-d 'no_of_guests=2' \
-d 'preferred_start_date=2024-04-01' \
-d 'message=Interested in the event' \
-d 'is_expecting_call_back=true'
```

### Notes

1. **Customer Profile Creation/Update**:
   - If a customer profile exists (matched by email or mobile), it will be updated
   - If no profile exists, a new one will be created

2. **Sales Representative Assignment**:
   - If an event is specified and has available sales representatives, one will be automatically assigned

3. **Notifications**:
   - A WhatsApp notification is automatically triggered upon successful enquiry creation

4. **Date Format**:
   - All dates should be provided in YYYY-MM-DD format
   - Invalid date formats will result in a 400 error

5. **Transaction Handling**:
   - All database operations are handled within a transaction
   - If any part fails, all changes will be rolled back

### Rate Limits
- Standard rate limiting applies
- Contact support for increased limits

### Support
For API support, contact: support@logout.world.com

This documentation provides a clear understanding of how to interact with the enquiry API endpoint, including all possible parameters, expected responses, and error handling scenarios.

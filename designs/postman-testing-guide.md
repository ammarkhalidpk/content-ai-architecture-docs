# Amazon Location Service - Postman Testing Guide

This guide provides ready-to-use Postman requests for testing the Address Autocomplete API using Amazon Location Service.

---

## Configuration

| Setting | Value |
|---------|-------|
| **Place Index Name** | `AddressAutocomplete` |
| **Place Index ARN** | `arn:aws:geo:ap-southeast-2:480399101976:place-index/AddressAutocomplete` |
| **Data Source** | Esri |
| **Region** | ap-southeast-2 (Sydney) |
| **API Key Expires** | December 31, 2026 |

---

## API Key

```
v1.public.eyJqdGkiOiI0MjIxYzVmMS03ZDk2LTQyYmQtOWQyZi04ODMyODI5NDE5Y2UifVz7glnV46xVrOrSUoHPTbPuzTVN1hd_3gJGWhWZCOJ3sipxqZYOVaBYmq0j43cg0mKwKKeMjcp3lZOQN45TerL3ewOL5R8R1ZthHv1gTAWsg_KKZ-yR6cO_3EnfkDIgDqvQTlFMF_reEP_WFqnsRosvNwnMDPF-PyHQAKosqEVE__1q5u82c9VfLlCJ3E7vyqkJsbQHkoiB9YQQl1fhOFpdrUSRE1_PVqgT_X6k_m-pd9lTIdVaJ0EtPEQV4DJCJfwl1ezaFORVkC_CBLT9EDl0jnsiMAup6EgxI-70AwEpj9d8vqwwbymZ1oqeioKWVg0UvPfr8J4uuQoG3PrrbjU.ZTA2OTdiZTItNzgyYy00YWI5LWFmODQtZjdkYmJkODNkMmFh
```

---

## Base URL

```
https://places.geo.ap-southeast-2.amazonaws.com
```

---

## Endpoints

### 1. Address Suggestions (Autocomplete)

Returns address suggestions as the user types. This is the primary endpoint for autocomplete functionality.

**Endpoint:**
```
POST /places/v0/indexes/AddressAutocomplete/search/suggestions
```

**Full URL with API Key:**
```
https://places.geo.ap-southeast-2.amazonaws.com/places/v0/indexes/AddressAutocomplete/search/suggestions?key=v1.public.eyJqdGkiOiI0MjIxYzVmMS03ZDk2LTQyYmQtOWQyZi04ODMyODI5NDE5Y2UifVz7glnV46xVrOrSUoHPTbPuzTVN1hd_3gJGWhWZCOJ3sipxqZYOVaBYmq0j43cg0mKwKKeMjcp3lZOQN45TerL3ewOL5R8R1ZthHv1gTAWsg_KKZ-yR6cO_3EnfkDIgDqvQTlFMF_reEP_WFqnsRosvNwnMDPF-PyHQAKosqEVE__1q5u82c9VfLlCJ3E7vyqkJsbQHkoiB9YQQl1fhOFpdrUSRE1_PVqgT_X6k_m-pd9lTIdVaJ0EtPEQV4DJCJfwl1ezaFORVkC_CBLT9EDl0jnsiMAup6EgxI-70AwEpj9d8vqwwbymZ1oqeioKWVg0UvPfr8J4uuQoG3PrrbjU.ZTA2OTdiZTItNzgyYy00YWI5LWFmODQtZjdkYmJkODNkMmFh
```

**Headers:**
```
Content-Type: application/json
```

**Request Body:**
```json
{
  "Text": "123 George St",
  "MaxResults": 5,
  "FilterCountries": ["AUS"]
}
```

**Example Response:**
```json
{
  "Results": [
    {
      "Text": "123 George Street, Sydney, NSW, 2000, AUS",
      "PlaceId": "AQAAAIAAkK..."
    },
    {
      "Text": "123 George Street, Parramatta, NSW, 2150, AUS",
      "PlaceId": "AQAAAIAAkL..."
    }
  ]
}
```

**cURL:**
```bash
curl -X POST \
  'https://places.geo.ap-southeast-2.amazonaws.com/places/v0/indexes/AddressAutocomplete/search/suggestions?key=v1.public.eyJqdGkiOiI0MjIxYzVmMS03ZDk2LTQyYmQtOWQyZi04ODMyODI5NDE5Y2UifVz7glnV46xVrOrSUoHPTbPuzTVN1hd_3gJGWhWZCOJ3sipxqZYOVaBYmq0j43cg0mKwKKeMjcp3lZOQN45TerL3ewOL5R8R1ZthHv1gTAWsg_KKZ-yR6cO_3EnfkDIgDqvQTlFMF_reEP_WFqnsRosvNwnMDPF-PyHQAKosqEVE__1q5u82c9VfLlCJ3E7vyqkJsbQHkoiB9YQQl1fhOFpdrUSRE1_PVqgT_X6k_m-pd9lTIdVaJ0EtPEQV4DJCJfwl1ezaFORVkC_CBLT9EDl0jnsiMAup6EgxI-70AwEpj9d8vqwwbymZ1oqeioKWVg0UvPfr8J4uuQoG3PrrbjU.ZTA2OTdiZTItNzgyYy00YWI5LWFmODQtZjdkYmJkODNkMmFh' \
  -H 'Content-Type: application/json' \
  -d '{
    "Text": "123 George St",
    "MaxResults": 5,
    "FilterCountries": ["AUS"]
  }'
```

---

### 2. Full Address Search (Geocoding)

Returns detailed place information with coordinates for a complete address search.

**Endpoint:**
```
POST /places/v0/indexes/AddressAutocomplete/search/text
```

**Full URL with API Key:**
```
https://places.geo.ap-southeast-2.amazonaws.com/places/v0/indexes/AddressAutocomplete/search/text?key=v1.public.eyJqdGkiOiI0MjIxYzVmMS03ZDk2LTQyYmQtOWQyZi04ODMyODI5NDE5Y2UifVz7glnV46xVrOrSUoHPTbPuzTVN1hd_3gJGWhWZCOJ3sipxqZYOVaBYmq0j43cg0mKwKKeMjcp3lZOQN45TerL3ewOL5R8R1ZthHv1gTAWsg_KKZ-yR6cO_3EnfkDIgDqvQTlFMF_reEP_WFqnsRosvNwnMDPF-PyHQAKosqEVE__1q5u82c9VfLlCJ3E7vyqkJsbQHkoiB9YQQl1fhOFpdrUSRE1_PVqgT_X6k_m-pd9lTIdVaJ0EtPEQV4DJCJfwl1ezaFORVkC_CBLT9EDl0jnsiMAup6EgxI-70AwEpj9d8vqwwbymZ1oqeioKWVg0UvPfr8J4uuQoG3PrrbjU.ZTA2OTdiZTItNzgyYy00YWI5LWFmODQtZjdkYmJkODNkMmFh
```

**Headers:**
```
Content-Type: application/json
```

**Request Body:**
```json
{
  "Text": "123 George Street Sydney NSW 2000",
  "MaxResults": 5,
  "FilterCountries": ["AUS"]
}
```

**Example Response:**
```json
{
  "Results": [
    {
      "Place": {
        "Label": "123 George Street, Sydney, NSW, 2000, AUS",
        "Geometry": {
          "Point": [151.2093, -33.8688]
        },
        "AddressNumber": "123",
        "Street": "George Street",
        "Municipality": "Sydney",
        "Region": "New South Wales",
        "SubRegion": "Sydney",
        "PostalCode": "2000",
        "Country": "AUS"
      },
      "PlaceId": "AQAAAIAAkK...",
      "Relevance": 1.0
    }
  ],
  "Summary": {
    "Text": "123 George Street Sydney NSW 2000",
    "DataSource": "Esri"
  }
}
```

**cURL:**
```bash
curl -X POST \
  'https://places.geo.ap-southeast-2.amazonaws.com/places/v0/indexes/AddressAutocomplete/search/text?key=v1.public.eyJqdGkiOiI0MjIxYzVmMS03ZDk2LTQyYmQtOWQyZi04ODMyODI5NDE5Y2UifVz7glnV46xVrOrSUoHPTbPuzTVN1hd_3gJGWhWZCOJ3sipxqZYOVaBYmq0j43cg0mKwKKeMjcp3lZOQN45TerL3ewOL5R8R1ZthHv1gTAWsg_KKZ-yR6cO_3EnfkDIgDqvQTlFMF_reEP_WFqnsRosvNwnMDPF-PyHQAKosqEVE__1q5u82c9VfLlCJ3E7vyqkJsbQHkoiB9YQQl1fhOFpdrUSRE1_PVqgT_X6k_m-pd9lTIdVaJ0EtPEQV4DJCJfwl1ezaFORVkC_CBLT9EDl0jnsiMAup6EgxI-70AwEpj9d8vqwwbymZ1oqeioKWVg0UvPfr8J4uuQoG3PrrbjU.ZTA2OTdiZTItNzgyYy00YWI5LWFmODQtZjdkYmJkODNkMmFh' \
  -H 'Content-Type: application/json' \
  -d '{
    "Text": "123 George Street Sydney NSW 2000",
    "MaxResults": 5,
    "FilterCountries": ["AUS"]
  }'
```

---

### 3. Get Place Details

Retrieves full details for a specific place using its PlaceId (obtained from suggestions or search).

**Endpoint:**
```
GET /places/v0/indexes/AddressAutocomplete/places/{PlaceId}
```

**Full URL with API Key:**
```
https://places.geo.ap-southeast-2.amazonaws.com/places/v0/indexes/AddressAutocomplete/places/{PlaceId}?key=v1.public.eyJqdGkiOiI0MjIxYzVmMS03ZDk2LTQyYmQtOWQyZi04ODMyODI5NDE5Y2UifVz7glnV46xVrOrSUoHPTbPuzTVN1hd_3gJGWhWZCOJ3sipxqZYOVaBYmq0j43cg0mKwKKeMjcp3lZOQN45TerL3ewOL5R8R1ZthHv1gTAWsg_KKZ-yR6cO_3EnfkDIgDqvQTlFMF_reEP_WFqnsRosvNwnMDPF-PyHQAKosqEVE__1q5u82c9VfLlCJ3E7vyqkJsbQHkoiB9YQQl1fhOFpdrUSRE1_PVqgT_X6k_m-pd9lTIdVaJ0EtPEQV4DJCJfwl1ezaFORVkC_CBLT9EDl0jnsiMAup6EgxI-70AwEpj9d8vqwwbymZ1oqeioKWVg0UvPfr8J4uuQoG3PrrbjU.ZTA2OTdiZTItNzgyYy00YWI5LWFmODQtZjdkYmJkODNkMmFh
```

**Example Response:**
```json
{
  "Place": {
    "Label": "123 George Street, Sydney, NSW, 2000, AUS",
    "Geometry": {
      "Point": [151.2093, -33.8688]
    },
    "AddressNumber": "123",
    "Street": "George Street",
    "Municipality": "Sydney",
    "Region": "New South Wales",
    "SubRegion": "Sydney",
    "PostalCode": "2000",
    "Country": "AUS",
    "Interpolated": false,
    "TimeZone": {
      "Name": "Australia/Sydney",
      "Offset": 39600
    }
  }
}
```

**cURL:**
```bash
curl -X GET \
  'https://places.geo.ap-southeast-2.amazonaws.com/places/v0/indexes/AddressAutocomplete/places/YOUR_PLACE_ID_HERE?key=v1.public.eyJqdGkiOiI0MjIxYzVmMS03ZDk2LTQyYmQtOWQyZi04ODMyODI5NDE5Y2UifVz7glnV46xVrOrSUoHPTbPuzTVN1hd_3gJGWhWZCOJ3sipxqZYOVaBYmq0j43cg0mKwKKeMjcp3lZOQN45TerL3ewOL5R8R1ZthHv1gTAWsg_KKZ-yR6cO_3EnfkDIgDqvQTlFMF_reEP_WFqnsRosvNwnMDPF-PyHQAKosqEVE__1q5u82c9VfLlCJ3E7vyqkJsbQHkoiB9YQQl1fhOFpdrUSRE1_PVqgT_X6k_m-pd9lTIdVaJ0EtPEQV4DJCJfwl1ezaFORVkC_CBLT9EDl0jnsiMAup6EgxI-70AwEpj9d8vqwwbymZ1oqeioKWVg0UvPfr8J4uuQoG3PrrbjU.ZTA2OTdiZTItNzgyYy00YWI5LWFmODQtZjdkYmJkODNkMmFh'
```

---

## Request Parameters

### Common Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `Text` | string | The search text (required) |
| `MaxResults` | integer | Maximum number of results (1-15, default: 5) |
| `FilterCountries` | array | ISO 3166-1 alpha-3 country codes (e.g., ["AUS"]) |
| `FilterBBox` | array | Bounding box [west, south, east, north] |
| `BiasPosition` | array | Bias results toward this location [longitude, latitude] |
| `Language` | string | Language code for results (e.g., "en") |

### Suggestions-Specific Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `FilterCategories` | array | Filter by place categories |

### Search-Specific Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `FilterCategories` | array | Filter by place categories |

---

## Example Request Bodies

### Basic Australian Address Search
```json
{
  "Text": "123 George St",
  "MaxResults": 5,
  "FilterCountries": ["AUS"]
}
```

### Search with Location Bias (Sydney CBD)
```json
{
  "Text": "George Street",
  "MaxResults": 10,
  "FilterCountries": ["AUS"],
  "BiasPosition": [151.2093, -33.8688]
}
```

### Search within Bounding Box (Sydney Area)
```json
{
  "Text": "Post Office",
  "MaxResults": 5,
  "FilterCountries": ["AUS"],
  "FilterBBox": [150.5, -34.2, 151.5, -33.5]
}
```

### Unit/Apartment Number Search
```json
{
  "Text": "Unit 5, 42 Oxford Street Darlinghurst",
  "MaxResults": 5,
  "FilterCountries": ["AUS"]
}
```

---

## Postman Collection Import

To import these requests into Postman:

1. Create a new Collection called "Amazon Location Service"
2. Add environment variable:
   - `api_key`: `v1.public.eyJqdGkiOiI0MjIxYzVmMS03ZDk2LTQyYmQtOWQyZi04ODMyODI5NDE5Y2UifVz7glnV46xVrOrSUoHPTbPuzTVN1hd_3gJGWhWZCOJ3sipxqZYOVaBYmq0j43cg0mKwKKeMjcp3lZOQN45TerL3ewOL5R8R1ZthHv1gTAWsg_KKZ-yR6cO_3EnfkDIgDqvQTlFMF_reEP_WFqnsRosvNwnMDPF-PyHQAKosqEVE__1q5u82c9VfLlCJ3E7vyqkJsbQHkoiB9YQQl1fhOFpdrUSRE1_PVqgT_X6k_m-pd9lTIdVaJ0EtPEQV4DJCJfwl1ezaFORVkC_CBLT9EDl0jnsiMAup6EgxI-70AwEpj9d8vqwwbymZ1oqeioKWVg0UvPfr8J4uuQoG3PrrbjU.ZTA2OTdiZTItNzgyYy00YWI5LWFmODQtZjdkYmJkODNkMmFh`
3. Use `{{api_key}}` in your request URLs

---

## Error Responses

### 400 Bad Request
```json
{
  "message": "Invalid request",
  "code": "ValidationException"
}
```

### 403 Forbidden
```json
{
  "message": "Access denied",
  "code": "AccessDeniedException"
}
```

### 429 Too Many Requests
```json
{
  "message": "Rate exceeded",
  "code": "ThrottlingException"
}
```

---

## Rate Limits

- **Suggestions**: 50 requests per second
- **Search**: 50 requests per second
- **GetPlace**: 50 requests per second

---

## Pricing

| Operation | Cost |
|-----------|------|
| SearchPlaceIndexForSuggestions | $0.50 per 1,000 requests |
| SearchPlaceIndexForText | $0.50 per 1,000 requests |
| GetPlace | $0.50 per 1,000 requests |

---

## Related Documentation

- [Amazon Location Service Design](/designs/amazon-location-service-design.md)
- [HERE Geocoding API Design](/designs/here-geocoding-api-design.md)
- [Design Comparison](/designs/design-comparison.md)

---

## Security Notes

- This API key has restricted permissions (suggestions, search, get place only)
- The key is restricted to the `AddressAutocomplete` place index only
- Key expires on December 31, 2026
- Rotate keys before expiration
- Do not expose this key in client-side code for production use

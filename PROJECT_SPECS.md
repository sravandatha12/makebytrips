# Project Specifications: Smart Destination Search & Hotel Management

## 1. Functional Requirements for Smart Destination Search

The "Smart Destination Search" is a specialized feature that enhances the hotel booking experience by suggesting accommodations based on their proximity to popular local attractions.

### User Interface (UI) Requirements
*   **Search Bar**: A prominent input field where users can type a city, region, or landmark (e.g., "Paris", "Eiffel Tower").
*   **Auto-Suggestions**: As the user types, the system should suggest cities and specific popular landmarks.
*   **Results Page Layout**:
    *   **"Top Attractions Nearby" Section**: A carousel or grid displaying top 3-5 tourist hotspots (e.g., "Louvre Museum", "Notre Dame") with images and distance from the searched location.
    *   **"Stay Near the Action" Section**: Hotel listings explicitly sorted/filtered by proximity to these attractions.
    *   **Interactive Map**: A map view pinning the searched location, top attractions (in one color), and nearby hotels (in another color).
*   **Smart Filters**: One-click filters for "Near City Center", "Near [Top Attraction]", "Quiet Neighborhoods".

### Functional Logic
1.  **Location Resolution**: When a user searches "Paris", the system identifies the center coordinates (lat/lng) of Paris.
2.  **Attraction Fetching**: The system queries an external database (or internal cache) for "top rated tourist attractions" within a 10-20km radius of the coordinates.
3.  **Proximity Calculation**: For each suggested hotel, calculate the distance to the identified top attractions.
4.  **Ranking Algorithm**: Boost hotels that are within walking distance (<2km) of high-popularity attractions.
5.  **Contextual Badges**: Display badges on hotel cards like "0.5km from Eiffel Tower" or "In the Heart of Old Town".

---

## 2. Database Schema (Relational/SQL Approach)

To link Hotels to Nearby Attractions efficiently, we can use a relational schema (e.g., PostgreSQL). We can either store attractions permanently or cache them. Below is a schema where we manage attractions as first-class entities.

### Tables

**1. Hotels**
*   `id` (UUID, PK)
*   `name` (VARCHAR)
*   `address` (TEXT)
*   `city` (VARCHAR)
*   `latitude` (DECIMAL)
*   `longitude` (DECIMAL)
*   `rating` (DECIMAL)
*   `base_price` (DECIMAL)

**2. Attractions**
*   `id` (UUID, PK)
*   `name` (VARCHAR) - e.g., "Eiffel Tower"
*   `type` (VARCHAR) - e.g., "Museum", "Park", "Landmark"
*   `city` (VARCHAR)
*   `latitude` (DECIMAL)
*   `longitude` (DECIMAL)
*   `popularity_score` (INT) - 1-100 scale based on external data/reviews
*   `image_url` (TEXT)

**3. Hotel_Attraction_Proximity (Junction Table)**
*   *Purpose: optimize queries so we don't calculate distance on every read.*
*   `hotel_id` (FK -> Hotels.id)
*   `attraction_id` (FK -> Attractions.id)
*   `distance_meters` (INT) - Pre-calculated distance
*   `walking_time_minutes` (INT) - Estimated walking time

### Relationships
*   **One Hotel** has **Many Nearby Attractions** (via Junction Table).
*   **One Attraction** has **Many Nearby Hotels** (via Junction Table).

### Query Logic
To find hotels near "Eiffel Tower":
```sql
SELECT h.*, hap.distance_meters
FROM Hotels h
JOIN Hotel_Attraction_Proximity hap ON h.id = hap.hotel_id
JOIN Attractions a ON hap.attraction_id = a.id
WHERE a.name = 'Eiffel Tower'
ORDER BY hap.distance_meters ASC;
```

---

## 3. API Endpoint Structure

### A. Fetch Popular Places
**Endpoint**: `GET /api/destinations/popular`

**Query Parameters**:
*   `city` (string, required): The city name to search (e.g., "London").
*   `lat` (float, optional): Latitude (if searching by "Current Location").
*   `lng` (float, optional): Longitude.
*   `radius` (int, optional): Search radius in meters (default: 5000).

**Response**:
```json
{
  "city": "London",
  "center": { "lat": 51.5074, "lng": -0.1278 },
  "attractions": [
    {
      "id": "attr_123",
      "name": "London Eye",
      "type": "Landmark",
      "rating": 4.5,
      "location": { "lat": 51.5033, "lng": -0.1195 },
      "image": "url_to_image",
      "description": "Giant Ferris wheel on the South Bank."
    },
    ...
  ]
}
```

### B. Search Hotels with Smart Context
**Endpoint**: `GET /api/hotels/search`

**Query Parameters**:
*   `location` (string): City or Landmark name.
*   `dates` (string): Date range.
*   `include_nearby_attractions` (boolean): If true, enriches response with proximity data.

**Response**:
```json
{
  "results": [
    {
      "hotel_id": "h_999",
      "name": "Riverside Plaza",
      "price": 250,
      "rating": 4.8,
      "location": { "lat": 51.5035, "lng": -0.1190 },
      "nearby_highlights": [
        {
          "name": "London Eye",
          "distance": "200m",
          "access": "3 min walk"
        }
      ]
    }
  ]
}
```

---

## 4. Third-Party API Integration (Recommendation)

To implement the "Smart Data" source:

1.  **Google Places API (New)**:
    *   **Pros**: Best coverage, rich photo content, accurate ratings.
    *   **Endpoints**: Text Search (find "top attractions in Paris"), Place Details.
    *   **Cost**: High at scale.

2.  **Mapbox Geocoding + POI**:
    *   **Pros**: Cheaper, good map visualizations.
    *   **Cons**: Less rich "tourist" metadata than Google.

3.  **Amadeus Travel Recommendations**:
    *   **Pros**: Designed specifically for travel (e.g., "Points of Interest" API).
    *   **Cons**: Specialized integration.

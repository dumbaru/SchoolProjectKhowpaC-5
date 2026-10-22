# Land Planning Tool (Nepal) — Frontend Requirements & API Spec (v1)

## 1. What we're building

A web app where a user draws their land plot on a satellite map, then plans what to do with it. Version 1 covers:

1. Account sign up / login
2. Draw a plot on the map and see its area in **ropani** (primary) and square metres (secondary)
3. **Subdivide** the plot: user says how many subplots, roads and facilities, and the system generates a layout
4. **Place buildings** (menu item only in v1; spec to follow, show as "Coming soon")
5. Save layouts to the account and **export a formal PDF**

The product is aimed at Nepal, so **mobile-first** design is important.

---

## 2. User flow

1. Sign up or log in
2. **My Plots** screen: list of saved plots, "New plot" button
3. **Map screen**: search a place, switch to satellite, draw the plot polygon, tap "Save plot"
4. **Plot details panel**: area shown (ropani big, m² small), with two actions: *Subdivide* and *Place buildings*
5. **Subdivision wizard**: enter number of subplots, roads, facilities, then tap Generate
6. **Layout result**: layout drawn on the map with a legend. Buttons: *Save*, *Export PDF*, *Regenerate*
7. Layout is view-only in v1 (no dragging or editing)

---

## 3. Screens and requirements

### 3.1 Sign up / Log in
- Fields: email, password (sign up also asks full name)
- On success, store the access token and send it on every request
- Show clear errors (wrong password, email already used)

### 3.2 My Plots
- List of plots: name, area in ropani, created date
- Tap a plot to open it on the map with its saved layouts listed
- "New plot" button, delete plot (with confirmation)

### 3.3 Map screen
- Satellite basemap (Esri World Imagery or Mapbox satellite), with a toggle to a street map
- Place search box (Nominatim or Mapbox geocoding)
- Draw a single polygon with Leaflet + leaflet-draw (or Leaflet-Geoman). Vertices must be editable after drawing
- Client-side checks before saving: at least 3 points, polygon must not cross itself
- Live area preview while drawing is optional, but the saved value always comes from the backend

### 3.4 Plot details panel
- Area: large text in ropani, e.g. **2-4-1-2** (ropani-aana-paisa-dam) plus the decimal value, small text in m²
- Two buttons: **Subdivide**, **Place buildings** (disabled with "Coming soon")
- Show the plot name, editable inline

### 3.5 Subdivision wizard
Fields:
| Field | Type | Rules |
|---|---|---|
| Number of subplots | integer | 2 to 50 |
| Number of roads | integer | 0 to 10 |
| Road width (m) | number | 3 to 20, default 6 |
| Facilities | list of `{type, count}` | types: `park`, `water_tank`, `community_hall`, `parking`, `temple`, `other` |

- Generate button shows a loading state (generation can take a few seconds)
- Show backend error messages inline

### 3.6 Layout result
- Draw the returned GeoJSON on the map, colour-coded by `kind`: subplot, road, facility
- Legend explaining colours
- Tap a subplot to show its label, area (ropani and m²) and dimensions
- Buttons: Save, Export PDF (download), Regenerate

### 3.7 General
- Responsive down to 360px width
- Loading, empty and error states for every screen
- English first. Keep all text in a translation file so Nepali can be added later

---

## 4. Tech suggestions (frontend)

- React (Vite) + TypeScript
- Leaflet + react-leaflet, with leaflet-draw or Leaflet-Geoman
- Turf.js for client-side polygon checks (self-intersection, rough area)
- Axios or fetch with a small API client wrapper

The backend is a REST API. Interactive docs will be available at `/docs` (Swagger) once the backend is running, so the frontend can test against it directly.

---

## 5. API conventions

- Base URL: `https://<host>/api/v1`
- Format: JSON, UTF-8
- Auth: `Authorization: Bearer <access_token>` on every endpoint except register and login
- **Coordinates are GeoJSON order: `[longitude, latitude]`**, CRS WGS84 (EPSG:4326)
- Timestamps: ISO 8601 UTC, e.g. `2026-10-22T08:30:00Z`
- IDs are UUID strings

### Error format (all errors)
```json
{
  "error": {
    "code": "INVALID_GEOMETRY",
    "message": "Polygon crosses itself.",
    "details": {}
  }
}
```

Common codes: `VALIDATION_ERROR` (422), `UNAUTHORIZED` (401), `FORBIDDEN` (403), `NOT_FOUND` (404), `INVALID_GEOMETRY` (422), `LAYOUT_INFEASIBLE` (422), `RATE_LIMITED` (429).

---

## 6. Shared objects

### Area object
```json
{
  "sq_m": 1273.4,
  "ropani": { "ropani": 2, "aana": 8, "paisa": 0, "dam": 2, "decimal": 2.5035 }
}
```
Conversion used by the backend: 1 ropani = 508.72 m², 1 ropani = 16 aana, 1 aana = 4 paisa, 1 paisa = 4 dam.

### Plot
```json
{
  "id": "b1c9...",
  "name": "Harry's land",
  "geometry": { "type": "Polygon", "coordinates": [[[85.324, 27.717], [85.325, 27.717], [85.325, 27.718], [85.324, 27.718], [85.324, 27.717]]] },
  "area": { "...": "Area object" },
  "perimeter_m": 145.2,
  "created_at": "2026-10-22T08:30:00Z",
  "updated_at": "2026-10-22T08:30:00Z"
}
```
Polygon rings must be closed (first point equals last point).

### Layout
```json
{
  "id": "c7d2...",
  "plot_id": "b1c9...",
  "params": {
    "subplot_count": 6,
    "road_count": 2,
    "road_width_m": 6,
    "facilities": [{ "type": "park", "count": 1 }]
  },
  "features": {
    "type": "FeatureCollection",
    "features": [
      {
        "type": "Feature",
        "geometry": { "type": "Polygon", "coordinates": [] },
        "properties": {
          "kind": "subplot",
          "label": "Plot 1",
          "area": { "...": "Area object" },
          "width_m": 18.2,
          "depth_m": 24.0
        }
      }
    ]
  },
  "summary": {
    "subplot_count": 6,
    "road_area_sq_m": 240.0,
    "facility_area_sq_m": 90.0,
    "usable_area_sq_m": 943.4
  },
  "created_at": "2026-10-22T08:35:00Z"
}
```
`properties.kind` is one of `subplot`, `road`, `facility`. Roads are returned as polygons (already buffered to the road width), so the frontend can draw them like any other shape.

---

## 7. Endpoints

### Auth

**POST `/auth/register`**
```json
// request
{ "email": "harry@example.com", "password": "min8chars", "full_name": "Harry Potter" }
// 201 response
{ "user": { "id": "u1...", "email": "harry@example.com", "full_name": "Harry Potter" },
  "access_token": "eyJ...", "token_type": "bearer" }
```

**POST `/auth/login`**
```json
// request
{ "email": "harry@example.com", "password": "min8chars" }
// 200 response
{ "access_token": "eyJ...", "token_type": "bearer", "expires_in": 3600 }
```

**GET `/auth/me`** returns the current user.

### Plots

| Method | Path | Description |
|---|---|---|
| POST | `/plots` | Create a plot |
| GET | `/plots` | List my plots (`?limit=20&offset=0`) |
| GET | `/plots/{plot_id}` | Get one plot |
| PATCH | `/plots/{plot_id}` | Rename or update geometry |
| DELETE | `/plots/{plot_id}` | Delete plot and its layouts (204) |

**POST `/plots`**
```json
// request
{ "name": "Harry's land", "geometry": { "type": "Polygon", "coordinates": [[[85.324, 27.717], ...]] } }
// 201 response: Plot object (area and perimeter computed by backend)
```
Rejected with `INVALID_GEOMETRY` if the polygon is self-intersecting, has fewer than 3 distinct points, or is under 10 m².

**GET `/plots`** response
```json
{ "items": [ { "...": "Plot object" } ], "total": 3, "limit": 20, "offset": 0 }
```

### Layouts (subdivision)

| Method | Path | Description |
|---|---|---|
| POST | `/plots/{plot_id}/layouts` | Generate a layout (not saved until confirmed) |
| POST | `/layouts/{layout_id}/save` | Save a generated layout to the account |
| GET | `/plots/{plot_id}/layouts` | List saved layouts for a plot |
| GET | `/layouts/{layout_id}` | Get one layout |
| DELETE | `/layouts/{layout_id}` | Delete a layout |

**POST `/plots/{plot_id}/layouts`**
```json
// request
{
  "subplot_count": 6,
  "road_count": 2,
  "road_width_m": 6,
  "facilities": [ { "type": "park", "count": 1 }, { "type": "water_tank", "count": 1 } ]
}
// 201 response: Layout object with "status": "generated"
```
If the parameters can't fit the plot (for example too many subplots for the area), the response is `LAYOUT_INFEASIBLE` with a human-readable message the frontend can show directly.

Calling this again with the same plot generates a new variation, which is what the Regenerate button uses.

### PDF export

**GET `/layouts/{layout_id}/export.pdf`**
- Response: `application/pdf` (download)
- Contents: title, plot name, date, plot area in ropani and m², scale bar and north arrow, layout drawing, table of subplots with dimensions and areas, road and facility summary
- Frontend: open in a new tab or trigger a download with the filename from the `Content-Disposition` header

### Place buildings (not in v1)
Endpoints to be defined later. Frontend should show the button disabled.

---

## 8. Auth and security notes for the frontend

- Store the token in memory or an httpOnly cookie if possible, avoid localStorage if the team can manage it
- On any `401`, clear the token and redirect to login
- Never trust client-side area numbers. Always display the values returned by the backend

---

## 9. Out of scope for v1

- Editing or dragging layout lines
- Importing government cadastral data
- Accuracy tools (surveyed reference points, GPS boundary walking, lalpurja measurements)
- Bigha-kattha-dhur units (Terai region), GeoJSON export, building placement

---

## 10. Open questions for the team

1. Which basemap provider will we use (Esri free tier or Mapbox with an API key)?
2. Does the PDF need any official header or format from the land office, or is a clean design enough for now?
3. Should the app support Nepali language at launch or only English?

# Geospatial File Measurement API

A production-oriented FastAPI backend that accepts KML files or ZIP archives containing Shapefiles, extracts geospatial features, handles CRS transformation, and calculates polygon areas and line lengths.

## Features

- FastAPI REST API
- KML upload
- ZIP-based Shapefile upload
- Feature extraction
- Polygon area calculation in square meters
- LineString length calculation in meters
- Point handling without measurement
- CRS-aware measurement calculation
- Automatic UTM CRS estimation for geographic CRS
- SQLite persistence for file metadata
- Graceful handling of unsupported geometries
- Swagger/OpenAPI documentation
- Pytest test suite
- Docker support

## Requirements

- Python 3.12+
- GDAL/Fiona dependencies if running outside Docker
- Git

## Local Setup

### 1. Clone repository

```bash
git clone <YOUR_GITHUB_REPOSITORY_URL>
cd geospatial-measurement-api
```

### 2. Create virtual environment

Windows:

```bash
python -m venv .venv
.venv\Scripts\activate
```

Linux/macOS:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

If your operating system has trouble installing Fiona/GDAL, use Docker.

### 4. Start API

```bash
uvicorn app.main:app --reload
```

API:
http://127.0.0.1:8000

Swagger:
http://127.0.0.1:8000/docs

## Docker

```bash
docker compose up --build
```

## API

### Upload

```http
POST /api/files/
Content-Type: multipart/form-data
```

Example:

```bash
curl -X POST "http://127.0.0.1:8000/api/files/" \
  -F "file=@survey.kml"
```

Response:

```json
{
  "id": "abc123",
  "filename": "survey.kml",
  "feature_count": 120,
  "crs": "EPSG:4326",
  "status": "COMPLETED"
}
```

For a Shapefile, upload a `.zip` containing at least `.shp`, `.shx`, `.dbf` and preferably `.prj`.

### File Information

```http
GET /api/files/{id}/
```

### Measurements

```http
GET /api/files/{id}/measurements/
```

Example:

```json
{
  "file_id": "abc123",
  "filename": "survey.kml",
  "feature_count": 2,
  "items": [
    {
      "feature_id": 0,
      "geometry_type": "Polygon",
      "properties": {
        "name": "Survey Area"
      },
      "measurement": {
        "area": 15432.67,
        "unit": "square_meters"
      },
      "status": "MEASURED"
    },
    {
      "feature_id": 1,
      "geometry_type": "LineString",
      "properties": {
        "name": "Road"
      },
      "measurement": {
        "length": 215.44,
        "unit": "meters"
      },
      "status": "MEASURED"
    }
  ]
}
```

## Architecture

```text
Client
  |
  v
FastAPI Router
  |
  +--> Validate file type
  |
  +--> Store uploaded file
  |
  +--> GeoPandas reads KML/Shapefile
  |
  +--> Persist metadata in SQLite
  |
  +--> Measurement Service
          |
          +--> Detect source CRS
          |
          +--> Estimate projected CRS
          |
          +--> Transform geometry
          |
          +--> Calculate area/length
  |
  v
JSON Response
```

## CRS Handling

Area and length must not be calculated directly from latitude/longitude coordinates because degrees are angular units.

If the source CRS is geographic, such as EPSG:4326, the application uses GeoPandas `estimate_utm_crs()` to select an appropriate local projected CRS.

For example:

```text
EPSG:4326
    |
    v
Estimate UTM CRS
    |
    v
Projected geometry
    |
    +--> Polygon area -> m²
    |
    +--> LineString length -> m
```

If the source data is already projected, its CRS is retained.

## Geometry Handling

| Geometry | Measurement |
|---|---|
| Polygon | Area |
| MultiPolygon | Area |
| LineString | Length |
| MultiLineString | Length |
| Point | None |
| Unsupported | Graceful status |

Unsupported geometries do not crash the complete measurement response.

## Design Decisions

### Why FastAPI?

FastAPI provides type-safe request/response schemas, automatic OpenAPI documentation, asynchronous file upload support, and good performance with a simple architecture.

### Why GeoPandas?

GeoPandas provides reliable geospatial file reading, CRS operations, geometry handling and integration with Shapely/PyProj.

### Why SQLite?

SQLite keeps the assignment simple and reproducible while still providing relational persistence. PostgreSQL + PostGIS would be a natural production upgrade.

### Why process synchronously?

The minimum assignment requires upload and processing but does not require asynchronous job tracking. Synchronous processing keeps the implementation simple. For very large files, the next step would be a background queue using Celery/RQ with Redis.

### Why UTM?

UTM is a practical projected CRS for local distance/area calculations and is usually much more appropriate than calculating directly in geographic degrees.

## Testing

Run:

```bash
pytest -v
```

The test suite covers health checking and upload validation. Additional geospatial integration tests can be added using fixture files generated with GDAL/Fiona.

## Security / Production Improvements

For production deployment, consider:

- Maximum upload size
- Virus/malware scanning
- Authentication and authorization
- Object storage such as S3
- PostgreSQL/PostGIS
- Background processing for large files
- Rate limiting
- Structured logging
- Metrics and tracing
- Stronger ZIP path/file validation
- Automatic cleanup and retention policies

## Learning

This project demonstrates practical backend development combined with geospatial data processing. Key learning areas include FastAPI API design, multipart file handling, GeoPandas, CRS transformation, Shapely geometry operations, database persistence, testing and Dockerization.

## Future Scope

- Async background processing
- PostGIS integration
- S3/object storage
- More geospatial formats
- Per-feature error reporting
- Authentication
- Pagination for large measurement responses
- Job progress tracking
- Improved CRS selection based on geometry extent
- Cloud deployment

## Submission Checklist

- [ ] Public GitHub repository
- [ ] README included
- [ ] Requirements included
- [ ] Tests included
- [ ] Docker configuration included
- [ ] Swagger endpoint verified
- [ ] KML upload tested
- [ ] Shapefile ZIP upload tested
- [ ] Measurement endpoint tested

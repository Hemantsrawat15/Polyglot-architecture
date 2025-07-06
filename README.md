# Polyglot-architecture

## How They Work Together
```
Satellite sends data → A microservice ingests it.

1. Images/videos saved to MinIO.

2. Metadata (e.g., satellite ID, timestamp, MinIO object link, geolocation) saved to PostgreSQL.

3. Sensor logs / JSON telemetry saved to MongoDB.

A query layer lets you combine metadata and telemetry when needed.
```
## What Does "Ingest" Mean?
Collecting and bringing data into a system for processing, storage, or analysis.

- That data ingestion system (or service) is responsible for:

- Uploading image to MinIO

- Storing metadata in PostgreSQL

- Storing telemetry in MongoDB

## How the data is linked to each other
### 1. PostgreSQL
Stores the master record:
```sql
satellite_metadata (
  id SERIAL PRIMARY KEY,
  satellite_id VARCHAR(50),
  timestamp TIMESTAMP,
  lat FLOAT,
  lon FLOAT,
  image_url TEXT  -- link to MinIO
)
```
Holds the main reference point.
The image_url field links directly to the MinIO image.
(satellite_id, timestamp) uniquely identifies the event.

### 2. MongoDB (telemetry)
Holds flexible, sensor-level data tied to the same satellite and timestamp:

```json
{
  "satellite_id": "SAT-123",
  "timestamp": "2025-07-06T12:00:00Z",
  "battery_level": 85,
  "orientation": { "x": 0.1, "y": 0.2, "z": 0.0 }
}
```
satellite_id + timestamp is the foreign key here.
You use this to match telemetry to metadata.

### 3. MinIO (images)
Holds the actual image files, named or organized using the same key:
```
/satellite-images/2025-07-06T12:00:00Z_SAT-123_image1.png
```
This is not a database, so there’s no schema — but we embed the same key in the object name or path and store that path in PostgreSQL.

## Example of Data to be stored in Databases

### Satellite Metadata -> PostgreSQL
``` json
{
  "satellite_id": "SAT-123",
  "timestamp": "2025-07-06T12:00:00Z",
  "lat": 37.7749,
  "lon": -122.4194
}
```
### 2. Telemetry Data -> MongoDB
``` json
{
  "satellite_id": "SAT-123",
  "timestamp": "2025-07-06T12:00:00Z",
  "battery_level": 87,
  "temperature": 22.5,
  "solar_panel_status": "OK",
  "orientation": {
    "x": 0.1,
    "y": 0.2,
    "z": -0.1
  },
  "errors": [],
  "payloads": [
    {
      "name": "thermal_camera",
      "status": "active"
    }
  ]
}
```

## Directory Structure (Microservice-style)
```
satellite-data-ingestion/
├── config.py              ← includes POSTGRESQL, MONGODB, MINIO dicts
├── db/
│   ├── postgresql_client.py
│   ├── mongodb_client.py
│   └── minio_client.py    ← optional helper if you prefer
├── ingest/
│   └── save_satellite_data.py
├── retrieve_satellite_data.py   ← now contains MinIO integration snippets
└── requirements.txt       ← psycopg2, pymongo, minio
```

## config.py
``` python
POSTGRESQL = {
    "host": "localhost",
    "port": 5432,
    "user": "postgres",
    "password": "postgres",
    "database": "satellite_data"
}

MONGODB = {
    "uri": "mongodb://localhost:27017/",
    "database": "satellite_data"
}

MINIO = {
    "endpoint": "localhost:9000",
    "access_key": "minioadmin",
    "secret_key": "minioadmin",
    "bucket": "satellite-images"
}

```

## db/postgresql_client.py
``` python
import psycopg2
from config import POSTGRESQL

def get_postgres_connection():
    return psycopg2.connect(
        host=POSTGRESQL['host'],
        port=POSTGRESQL['port'],
        user=POSTGRESQL['user'],
        password=POSTGRESQL['password'],
        database=POSTGRESQL['database']
    )

def save_metadata(metadata):
    conn = get_postgres_connection()
    cursor = conn.cursor()
    cursor.execute("""
        INSERT INTO satellite_metadata (satellite_id, timestamp, lat, lon, image_url)
        VALUES (%s, %s, %s, %s, %s)
    """, (metadata['satellite_id'], metadata['timestamp'], metadata['lat'], metadata['lon'], metadata['image_url']))
    conn.commit()
    cursor.close()
    conn.close()
```

## db/mongodb_client.py
``` python
from pymongo import MongoClient
from config import MONGODB

client = MongoClient(MONGODB['uri'])
db = client[MONGODB['database']]

def save_telemetry(telemetry):
    db.telemetry.insert_one(telemetry)
```

## db/minio_client.py
``` python
from minio import Minio
from config import MINIO
import datetime

client = Minio(
    MINIO['endpoint'],
    access_key=MINIO['access_key'],
    secret_key=MINIO['secret_key'],
    secure=False
)

def upload_image(file_path, object_name=None):
    if not client.bucket_exists(MINIO['bucket']):
        client.make_bucket(MINIO['bucket'])

    if object_name is None:
        object_name = f"{datetime.datetime.utcnow().isoformat()}_{file_path.split('/')[-1]}"

    client.fput_object(MINIO['bucket'], object_name, file_path)
    return f"http://{MINIO['endpoint']}/{MINIO['bucket']}/{object_name}"
```

## ingest/save_satellite_data.py
``` python
import json
from db.postgresql_client import save_metadata
from db.mongodb_client import save_telemetry
from db.minio_client import upload_image

def ingest_data(image_path, metadata, telemetry):
    # 1. Upload image to MinIO
    image_url = upload_image(image_path)
    
    # 2. Save metadata to PostgreSQL
    metadata['image_url'] = image_url
    save_metadata(metadata)

    # 3. Save telemetry to MongoDB
    telemetry['satellite_id'] = metadata['satellite_id']
    telemetry['timestamp'] = metadata['timestamp']
    save_telemetry(telemetry)

# Example usage
if __name__ == "__main__":
    image_path = "images/sat_image_123.png"
    
    metadata = {
        "satellite_id": "SAT-123",
        "timestamp": "2025-07-06T12:00:00Z",
        "lat": 37.7749,
        "lon": -122.4194
    }

    telemetry = {
        "battery_level": 85,
        "temperature": 22.5,
        "orientation": {"x": 0.1, "y": 0.3, "z": 0.0}
    }

    ingest_data(image_path, metadata, telemetry)
```

## Retreiving the data
```python
import psycopg2
from pymongo import MongoClient
from minio import Minio
from datetime import timedelta
from config import POSTGRESQL, MONGODB, MINIO


def get_postgres_connection():
    return psycopg2.connect(
        host=POSTGRESQL["host"],
        port=POSTGRESQL["port"],
        user=POSTGRESQL["user"],
        password=POSTGRESQL["password"],
        database=POSTGRESQL["database"]
    )


def get_mongo_client():
    client = MongoClient(MONGODB["uri"])
    return client[MONGODB["database"]]


def get_minio_client():
    return Minio(
        MINIO["endpoint"],
        access_key=MINIO["access_key"],
        secret_key=MINIO["secret_key"],
        secure=MINIO.get("secure", False)
    )


def retrieve_satellite_data(satellite_id: str, timestamp: str):
    """
    Fetch data from PostgreSQL (metadata + image_url), MongoDB (telemetry),
    and MinIO (generate signed URL for image), then return unified view.
    """
    result = {}

    # Step 1: Get metadata and image_url from PostgreSQL
    try:
        conn = get_postgres_connection()
        cursor = conn.cursor()
        cursor.execute("""
            SELECT satellite_id, timestamp, lat, lon, image_url
            FROM satellite_metadata
            WHERE satellite_id = %s AND timestamp = %s
        """, (satellite_id, timestamp))
        row = cursor.fetchone()
        cursor.close()
        conn.close()

        if not row:
            return {"error": "No metadata found for given satellite_id and timestamp."}

        result["metadata"] = {
            "satellite_id": row[0],
            "timestamp": row[1].isoformat(),
            "lat": row[2],
            "lon": row[3],
            "image_url": row[4]
        }

    except Exception as e:
        return {"error": f"PostgreSQL error: {str(e)}"}

    # Step 2: Get telemetry from MongoDB
    try:
        db = get_mongo_client()
        telemetry_doc = db.telemetry.find_one({
            "satellite_id": satellite_id,
            "timestamp": timestamp
        })

        result["telemetry"] = telemetry_doc if telemetry_doc else {}
    except Exception as e:
        result["telemetry"] = {"error": f"MongoDB error: {str(e)}"}

    # Step 3: Generate a pre-signed URL from MinIO (optional)
    try:
        minio_client = get_minio_client()

        # Extract object name from the URL
        if result["metadata"].get("image_url"):
            # image_url = http://localhost:9000/satellite-images/filename.png
            object_path = result["metadata"]["image_url"].split(f"/{MINIO['bucket']}/")[-1]

            presigned_url = minio_client.presigned_get_object(
                bucket_name=MINIO["bucket"],
                object_name=object_path,
                expires=timedelta(minutes=15)
            )
            result["metadata"]["image_signed_url"] = presigned_url
        else:
            result["metadata"]["image_signed_url"] = None
    except Exception as e:
        result["metadata"]["image_signed_url"] = None
        result["metadata"]["signed_url_error"] = str(e)

    return result
```

## Final output
```json
{
  "metadata": {
    "satellite_id": "SAT-123",
    "timestamp": "2025-07-06T12:00:00Z",
    "lat": 37.7749,
    "lon": -122.4194,
    "image_url": "http://localhost:9000/satellite-images/SAT-123_20250706_120000.png",
    "image_signed_url": "http://localhost:9000/satellite-images/...signed_url_here..."
  },
  "telemetry": {
    "_id": "...",
    "satellite_id": "SAT-123",
    "timestamp": "2025-07-06T12:00:00Z",
    "battery_level": 85,
    "orientation": {
      "x": 0.1,
      "y": 0.2,
      "z": 0.0
    }
  }
}
```
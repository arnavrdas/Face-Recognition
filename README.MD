# Face Recognition API

A FastAPI-based identity verification service for dating apps.
Compares reference photos against multiple selfie angles to confirm it's the same person using **Facenet512** + **RetinaFace**.

---

## Table of Contents

- [Project Structure](#project-structure)
- [Tech Stack](#tech-stack)
- [Local Setup](#local-setup)
- [API Endpoints](#api-endpoints)
- [Request & Response Format](#request--response-format)
- [Configuration](#configuration)
- [Error Handling](#error-handling)
- [App Flow](#app-flow)
- [Tips for Better Accuracy](#tips-for-better-accuracy)

---

## Project Structure

```
face_recognition_api/
├── main.py             # FastAPI app entry point, CORS, lifespan
├── routes.py           # API endpoints
├── service.py          # Core logic: RetinaFace + Facenet512 + cosine distance
├── schemas.py          # Pydantic models for request/response
├── config.py           # All thresholds and settings
├── requirements.txt    # Python dependencies
└── README.md           # This file
```

---

## Tech Stack

| Component | Library |
|-----------|---------|
| API Framework | FastAPI |
| Face Detection | RetinaFace |
| Face Recognition | DeepFace (Facenet512) |
| Image Processing | OpenCV |
| Distance Metric | Cosine similarity |

---

## Local Setup

### 1. Clone the repository

```bash
git clone <your-repo-url>
cd face_recognition_api
```

### 2. Create and activate virtual environment

```bash
# Windows
python -m venv venv
venv\Scripts\activate

# Mac / Linux
python -m venv venv
source venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

> First install will take a few minutes. TensorFlow is ~500MB.
> The Facenet512 model weights (~90MB) will auto-download on the very first API request.

### 4. Run the server

```bash
python main.py
```

Server runs at: `http://localhost:8000`
Interactive API docs: `http://localhost:8000/docs`

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/verify` | Full verification — 1–2 reference images + 2–6 selfies |
| POST | `/api/v1/verify/quick` | Quick 1-vs-1 face comparison |
| GET | `/` | Service info |
| GET | `/health` | Health check |

---

## Request & Response Format

### POST `/api/v1/verify`

**Request type:** `multipart/form-data`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reference_images` | File(s) | Yes | 1–2 reference photos (JPG/PNG, max 10MB each) |
| `selfie_images` | File(s) | Yes | 2–6 selfie photos (JPG/PNG, max 10MB each) |

**Success Response — `200 OK`**

```json
{
  "status": "verified",
  "confidence_score": 0.875,
  "matched_pairs": 10,
  "total_pairs": 12,
  "match_ratio": 0.83,
  "pair_results": [
    {
      "reference_index": 0,
      "selfie_index": 0,
      "is_match": true,
      "confidence_score": 0.87,
      "distance": 0.195
    }
  ],
  "message": "Identity verified. 10/12 image pairs matched with 87.5% confidence."
}
```

**Response fields explained**

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | `verified` or `rejected` |
| `confidence_score` | float | Overall confidence 0.0–1.0 |
| `matched_pairs` | int | Number of pairs that matched |
| `total_pairs` | int | Total pairs compared |
| `match_ratio` | float | matched / total |
| `pair_results` | array | Per-pair breakdown with distance and score |
| `message` | string | Human-readable result summary |

---

### POST `/api/v1/verify/quick`

Quick 1-vs-1 comparison. Same response format as above.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reference_image` | File | Yes | Single reference photo (JPG/PNG) |
| `selfie_image` | File | Yes | Single selfie photo (JPG/PNG) |

---

## Configuration

All settings are in `config.py` and can be overridden via a `.env` file.

| Variable | Default | Description |
|----------|---------|-------------|
| `RECOGNITION_MODEL` | `Facenet512` | Face embedding model |
| `DETECTOR_BACKEND` | `retinaface` | Face detection backend |
| `MATCH_THRESHOLD` | `0.40` | Cosine distance cutoff per pair (lower = stricter) |
| `MIN_MATCH_RATIO` | `0.50` | Minimum % of pairs that must match for verified result |
| `MAX_FILE_SIZE_MB` | `10` | Max image size per uploaded file |
| `MAX_SELFIE_IMAGES` | `6` | Max number of selfie images allowed |

---

## Error Handling

| HTTP Code | Meaning | Cause |
|-----------|---------|-------|
| `400` | Bad Request | Wrong file type, file too large, or wrong image count |
| `422` | Unprocessable | No face detected in one or more images |
| `500` | Server Error | Internal model or embedding failure |

All error responses follow this format:

```json
{
  "error": "string",
  "detail": "string"
}
```

---

## App Flow

This is the intended user flow for the dating app:

```
Step 1 — User picks 1–2 photos from their device gallery
              ↓
Step 2 — App opens camera, user captures 4–6 selfies
         at different angles (front, left, right, up, down, smile)
              ↓
Step 3 — App sends all images to POST /api/v1/verify
              ↓
Step 4 — API returns verified or rejected with confidence score
              ↓
Step 5 — App shows result to the user
```

**Important notes for mobile integration:**
- Send images as `multipart/form-data` — not base64, not JSON
- Use field name `reference_images` for gallery photos
- Use field name `selfie_images` for camera captures
- Set request timeout to at least 60 seconds — the first request loads the model
- Full body photos are supported — the API automatically detects and crops the face

---

## Tips for Better Accuracy

- Reference photos should clearly show the face — avoid group photos or far-away shots
- Selfie angles to capture: **front, left, right, up, down, smile**
- Keep angle tilts subtle — extreme angles (fully sideways) reduce match accuracy
- Good lighting matters — faces lit from the front perform best
- Avoid backlighting (e.g. sitting in front of a bright window)
- Minimum face size in any image should be at least 80×80 pixels for reliable detection
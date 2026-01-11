# Number Plate Detector API — PoC Requirements (requirements.md)

## 1. Purpose
Build a Dockerized API that accepts a vehicle image (upload or URL) and returns:
- recognized number plate text (`plate_text`)
- derived state code/name (`plate_state`, `plate_state_name`) when possible
- standardized status + confidence
- standardized error codes when detection/OCR fails

This is a **1-week PoC** for college students. Accuracy is expected to be good on the provided sample set, not perfect on every real-world scenario.

---

## 2. Target Environment
- Development may be done on **Windows**
- Deployment must run on **Linux (Ubuntu)** via **Docker**
- Solution must be runnable using:
  - `docker compose up --build`

**No host-machine installs required** (no “pip install on Windows and run locally” without Docker).

---

## 3. Scope

### In-scope (PoC)
- Plate detection + OCR pipeline using any approach (AI models, classical CV, etc.)
- Standard JSON response format for all outcomes
- Confidence score (0–100)
- Error codes for common failure modes
- Docker + Ubuntu compatibility
- Batch test script + sample images

### Out-of-scope (PoC)
- Training custom models from scratch
- GPU/CUDA requirement (CPU-only by default)
- Production hardening: auth, rate limits, autoscaling, distributed tracing, etc.
- Guaranteed performance on night/rain/motion blur/crazy angles (unless included in sample set)

---

## 4. API Requirements

### 4.1 Endpoints

#### `GET /health`
Returns service health.

**Response**
```json
{ "status": "ok" }


POST /v1/plate/detect

Accepts an image via either upload or URL and returns detection/OCR results.

4.2 Input Modes
Mode A: Upload

Content-Type: multipart/form-data

Fields:

image (required): jpg/png/webp

source_id (optional): string (caller’s reference id)

return_candidates (optional): boolean (true|false)

Mode B: URL fetch

Content-Type: application/json

Body:
{
  "image_url": "https://example-bucket/path/car.jpg",
  "source_id": "optional-id",
  "return_candidates": true
}
5. Response Contract (Mandatory)
5.1 Success Response

Must return status: "ok"

Must return confidence as integer 0–100

Must return result.plate_text non-empty

plate_state may be null if unknown
{
  "status": "ok",
  "confidence": 90,
  "result": {
    "plate_text": "KA01AB1234",
    "plate_state": "KA",
    "plate_state_name": "Karnataka",
    "bbox": { "x": 312, "y": 420, "w": 280, "h": 90 }
  },
  "candidates": [
    {
      "plate_text": "KA01AB1234",
      "confidence": 90,
      "bbox": { "x": 312, "y": 420, "w": 280, "h": 90 }
    }
  ],
  "meta": {
    "request_id": "uuid-or-random-id",
    "processing_ms": 412,
    "engine": "detector+ocr",
    "version": "1.0.0"
  }
}

5.2 Failure Response

Must return status: "fail"

Must include error.code from the allowed list

Must include error.message

Must include meta.request_id

{
  "status": "fail",
  "confidence": 0,
  "error": {
    "code": "NO_NUMBER_PLATE",
    "message": "No number plate found in the image.",
    "details": { "hint": "Try a clearer photo." }
  },
  "meta": {
    "request_id": "uuid-or-random-id",
    "processing_ms": 210,
    "engine": "detector+ocr",
    "version": "1.0.0"
  }
}
6. Error Codes (Mandatory)

At minimum, implement these:

NO_NUMBER_PLATE — plate not detected

UNCLEAR — plate detected but OCR unreliable/unreadable

LOW_CONFIDENCE — detected/OCR confidence below thresholds

INVALID_IMAGE — unsupported/corrupt image

IMAGE_TOO_LARGE — exceeds configured limit

URL_FETCH_FAILED — cannot download from URL

TIMEOUT — processing exceeded time limit

INTERNAL_ERROR — unexpected failure

7. Confidence Rules

confidence must be 0–100.

Recommended approach: combine detection confidence + OCR confidence.

Include thresholds (configurable):

MIN_DET_CONF (default 0.40)

MIN_OCR_CONF (default 0.50)

If a plate is detected but OCR confidence is too low:

return status: fail

error.code: "UNCLEAR" or "LOW_CONFIDENCE"

8. Plate Text Normalization

Output plate_text must be normalized:

uppercase

trim whitespace

remove obvious non-plate characters (configurable)

optional: validate using regex patterns (based on target region)

9. State Extraction (Configurable)

If the project targets India plates:

derive plate_state from first 2 letters (e.g., KA, MH, DL)

map to plate_state_name using a config file, e.g. state_codes.json

If state cannot be derived:

return plate_state: null

do not fail if plate_text exists

10. Docker + Linux Requirements (Mandatory)

Deliverables must include:

Dockerfile

docker-compose.yml

pinned dependencies:

Python: requirements.txt

Node: package-lock.json

Constraints:

Must run on Ubuntu host with Docker installed

No hard-coded Windows paths

CPU-only by default

11. Logging (Minimum)

Log one structured line per request with:

request_id

status

confidence

error.code (if fail)

processing_ms

12. Testing Requirements

Provide:

samples/ folder:

at least 10 images with clear plates (expected success)

at least 5 images with no plates (expected NO_NUMBER_PLATE)

at least 5 blurry/occluded plates (expected UNCLEAR or LOW_CONFIDENCE)

test_client.py (or equivalent) that runs the API against samples/ and prints JSON results.

13. Performance (PoC Targets)

Average processing time target (CPU): < 1.5s per image on typical images (best effort)

Hard timeout: 8s per request (configurable)

14. Documentation (Mandatory)

Include a README.md with:

How to run:

docker compose up --build

API usage examples (curl)

Environment variables and defaults

Known limitations

15. Example curl (single-line)

Note: If TLS is not terminated in the container, the caller may replace https with http in their environment.

Upload

curl -s -X POST "https://localhost:8080/v1/plate/detect" -F "image=@./samples/car1.jpg" -F "source_id=test1" -F "return_candidates=true"
URL
curl -s -X POST "https://localhost:8080/v1/plate/detect" -H "Content-Type: application/json" -d "{\"image_url\":\"https://example-bucket/path/car1.jpg\",\"source_id\":\"test2\",\"return_candidates\":true}"


16. Acceptance Criteria (Must Pass)

docker compose up --build starts service successfully on Ubuntu.

GET /health returns { "status": "ok" }.

POST /v1/plate/detect (upload) returns valid JSON in both success and fail cases.

Response always includes status and meta.request_id.

Failure responses always include error.code from the defined list.

Test script runs on the provided samples/ and prints outputs without manual steps.

17. Recommended Architecture (Not Mandatory)

Python FastAPI service (best for CV/OCR) OR

Node.js gateway calling a Python worker service

Keep this as a microservice so it can be integrated by HTTP from other systems.


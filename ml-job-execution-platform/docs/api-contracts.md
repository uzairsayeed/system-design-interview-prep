# API Contracts

## Submit job

```http
POST /v1/jobs
Idempotency-Key: 9a7...
Authorization: Bearer ...
```

```json
{
  "priority": "HIGH",
  "image": "registry.example.com/ml/trainer:v4",
  "command": ["python", "train.py"],
  "environment": {
    "DATASET_URI": "object://datasets/train/"
  },
  "resources": {
    "cpu_millis": 8000,
    "memory_mb": 32768,
    "gpu_count": 1,
    "gpu_type": "A100"
  },
  "input_uri": "object://datasets/train/",
  "output_uri": "object://outputs/job-123/",
  "max_runtime_seconds": 21600
}
```

Responses:

- `201 Created` — new job accepted
- `200 OK` — matching idempotent request already exists
- `409 Conflict` — idempotency key reused with a different payload
- `422 Unprocessable Entity` — invalid resource specification
- `429 Too Many Requests` — tenant admission limit exceeded
- `503 Service Unavailable` — platform temporarily unable to accept work

## Get job

```http
GET /v1/jobs/{job_id}
```

## List jobs

```http
GET /v1/jobs?status=RUNNING&limit=50&cursor=...
```

## Cancel job

```http
POST /v1/jobs/{job_id}/cancel
```

Responses:

- `200 OK` — pending job cancelled synchronously
- `202 Accepted` — running job moved to `CANCELLING`
- `409 Conflict` — terminal job cannot be cancelled

## Worker assignment poll

```http
POST /v1/workers/{worker_id}/assignments/poll?wait_seconds=30
```

## Worker heartbeat

```http
POST /v1/workers/{worker_id}/heartbeat
```

## Attempt started

```http
POST /v1/workers/{worker_id}/attempts/{attempt_id}/started
```

## Attempt completed

```http
POST /v1/workers/{worker_id}/attempts/{attempt_id}/completed
```

## Attempt failed

```http
POST /v1/workers/{worker_id}/attempts/{attempt_id}/failed
```

## Attempt cancelled

```http
POST /v1/workers/{worker_id}/attempts/{attempt_id}/cancelled
```

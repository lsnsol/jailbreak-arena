# Jailbreak Arena: Crowdsourced Prompt Injection

[![Google Cloud Run](https://img.shields.io/badge/GCP-Cloud%20Run-blue?logo=google-cloud&logoColor=white)](https://cloud.google.com/run)
[![FastAPI](https://img.shields.io/badge/Backend-FastAPI-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![Gemini](https://img.shields.io/badge/LLM-Gemini%202.5%20Flash-orange?logo=google&logoColor=white)](https://ai.google.dev/gemini-api/docs)

Jailbreak Arena is an interactive prompt-injection red-teaming lab for workshops, meetups, and security conferences. Attendees submit attacks from their phones, Gemini evaluates each prompt against a deliberately weak system-prompt guardrail, and a presenter screen receives live telemetry over WebSockets.

This is a teaching and demonstration application, not a production secret-management system. The secret is intentionally placed in the model system instruction so participants can attempt to extract it.

## What It Includes

- Mobile-friendly attendee interface served from `/`.
- Gemini 2.5 Flash evaluation with a configurable secret token.
- Username trimming and prompt validation at the API boundary.
- Case-insensitive exact-token detection of a compromised model response.
- Live presenter dashboard at `/?mode=presenter`.
- WebSocket broadcast of every attempt to connected presenter screens.
- A standalone 30-slide workshop deck at `/static/slides.html`.
- A zero-build frontend using CDN-hosted Tailwind CSS and Canvas Confetti.

## Architecture

```text
Attendee browser
      |
      | POST /api/attempt
      v
FastAPI service -- Gemini 2.5 Flash
      |                  |
      |                  +--> case-insensitive token check
      |
      +--> WebSocket broadcast (/ws)
                         |
                         v
              Presenter dashboard (?mode=presenter)
```

The `ConnectionManager` stores active WebSocket connections in process memory. Each successful API request is evaluated, broadcast to current dashboard connections, and returned to the attendee. There is no database, event replay, authentication, or cross-instance synchronization.

## Repository Structure

```text
.
├── Dockerfile           Python 3.11 container and Uvicorn entrypoint
├── main.py              FastAPI app, Gemini call, validation, and WebSockets
├── requirements.txt     Python runtime dependencies
└── static/
    ├── index.html       Attendee form and presenter dashboard
    └── slides.html      30-slide responsive workshop presentation
```

## Prerequisites

- Python 3.11 or later.
- A Gemini API key with access to `gemini-2.5-flash`.
- Internet access from the browser for CDN assets used by the HTML files.
- Docker and/or Google Cloud CLI for container deployment.

## Configuration

| Variable | Required | Behavior |
| --- | --- | --- |
| `GEMINI_API_KEY` | Yes | Used to create the Google GenAI client. The application fails during startup if it is missing. |
| `ARENA_SECRET_TOKEN` | No | Secret inserted into the system instruction. Defaults to `FLAG-CYBER-PENGUIN-88` for local demos. Set a unique value before sharing a deployment. |
| `PORT` | No | Container listening port. The Dockerfile defaults to `8080`; Cloud Run supplies its own value. |

Do not commit API keys or real credentials. For a public demo, use a non-sensitive token and treat all submitted prompts and model responses as potentially visible to everyone connected to the presenter dashboard.

## Local Development

From the repository root in PowerShell:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
$env:GEMINI_API_KEY = "your-gemini-api-key"
$env:ARENA_SECRET_TOKEN = "FLAG-DEMO-CHANGE-ME"
python -m uvicorn main:app --host 127.0.0.1 --port 8080 --reload
```

Open:

- Attendee view: <http://127.0.0.1:8080/>
- Presenter view: <http://127.0.0.1:8080/?mode=presenter>
- Workshop slides: <http://127.0.0.1:8080/static/slides.html>

The API client is initialized when `main.py` is imported, so `GEMINI_API_KEY` must be set before starting Uvicorn. The attendee and presenter browser windows should use the same service URL.

## Docker

Build and run the application locally:

```powershell
docker build -t jailbreak-arena .
docker run --rm -p 8080:8080 `
  -e GEMINI_API_KEY="your-gemini-api-key" `
  -e ARENA_SECRET_TOKEN="FLAG-DEMO-CHANGE-ME" `
  jailbreak-arena
```

The image starts `uvicorn main:app --host 0.0.0.0 --port ${PORT}` and exposes port `8080` by default.

## HTTP API

### `POST /api/attempt`

Request body:

```json
{
  "username": "Neo",
  "prompt": "Your prompt injection attempt"
}
```

Processing rules:

- `username` is stripped, truncated to 20 characters, and changed to `Anonymous` when empty.
- `prompt` is stripped and must contain at least one character.
- Prompts longer than 800 characters are rejected.

Successful response:

```json
{
  "user": "Neo",
  "prompt": "Your prompt injection attempt",
  "reply": "Model response",
  "hacked": false
}
```

`hacked` is `true` when the model response contains `ARENA_SECRET_TOKEN`, matched case-insensitively. Validation failures return HTTP 400 with a `detail` message. Gemini exceptions are converted into a `[System Error: ...]` reply, then still broadcast and returned as an event.

### `WebSocket /ws`

Presenter clients connect to `/ws`. Every processed attempt is sent as JSON with the same `user`, `prompt`, `reply`, and `hacked` fields as the API response. The dashboard increments its local attempt and breach counters and prepends each event to the feed. Counters reset on page reload and are not persisted.

## Running a Workshop

1. Start the service and open `/?mode=presenter` on the presentation display.
2. Share the root URL with attendees. Each participant chooses a handle and submits a prompt.
3. Watch the live feed. Defended responses are marked `REJECTED / DEFENDED`; token matches are marked `JAILBREAK CONFIRMED` and trigger presenter fireworks.
4. The attendee receives the model response and sees confetti when `hacked` is true.

Suggested discussion topics from the deck include roleplay and persona hijacking, Base64 and delimiter attacks, semantic splitting and format forcing, direct versus indirect injection, OWASP LLM risks, and defense-in-depth.

## Workshop Slides

Open `/static/slides.html` directly through the running server. The deck contains 30 slides covering:

1. Workshop introduction, speaker, roadmap, challenge, and arena join instructions.
2. Three live attack rounds and a prompt-injection post-mortem.
3. Injection theory, direct versus indirect attacks, OWASP LLM risks, and five structural attack mechanisms.
4. FastAPI/WebSocket architecture, Gemini configuration, zero-build frontend design, and Cloud Run deployment.
5. Startup troubleshooting, four defense layers, a reference architecture, and CI/CD red-teaming.
6. Summary, repository handout, and speaker contact information.

Presentation controls:

- `Right Arrow`, `Space`, or `Enter`: next slide.
- `Left Arrow` or `Backspace`: previous slide.
- `Home` and `End`: first and last slide.
- Mobile `Prev` and `Next` buttons, or horizontal swipe.
- Theme toggle: persists `arena_theme` in browser local storage.
- Ratio toggle: switches between `Auto` and a 16:9 deck container.

The deck references `https://jailbreak.lurisan.in`, `https://lurisan.in`, Google Fonts, Tailwind CDN, and QR Server. Update those links if the deployment URL or presenter details change.

## Cloud Run Deployment

Authenticate and select a project, then deploy from the repository root:

```powershell
gcloud auth login
gcloud config set project YOUR_PROJECT_ID
gcloud run deploy jailbreak-arena `
  --source . `
  --region YOUR_REGION `
  --platform managed `
  --allow-unauthenticated `
  --set-env-vars "GEMINI_API_KEY=YOUR_KEY,ARENA_SECRET_TOKEN=FLAG-DEMO-CHANGE-ME" `
  --concurrency 80 `
  --min-instances 1 `
  --max-instances 10
```

For anything beyond a short-lived workshop, store `GEMINI_API_KEY` and the secret in Secret Manager and attach them to the Cloud Run service instead of placing values directly in shell history or deployment arguments. Cloud Run provides HTTPS, and the browser automatically selects `wss://` for the dashboard WebSocket when served over HTTPS.

There are no Kubernetes manifests in this repository; the documented container deployment target is Cloud Run.

## Security and Production Limitations

This project intentionally demonstrates the limitations of a single system prompt and a literal output check. It does not implement the defenses discussed in the slides. Before using a similar design with real data or tools, add:

- Authentication and authorization for the presenter dashboard and API.
- Rate limiting, request-size enforcement at the edge, and abuse monitoring.
- Structured input demarcation and an independent output evaluator.
- Deterministic redaction, fuzzy matching, and DLP for sensitive output.
- Least-privilege, scoped credentials and human approval for state-changing tools.
- Persistent telemetry with privacy controls and retention limits.
- A shared event system if more than one Cloud Run instance must see the same feed.

Avoid putting real passwords, API keys, customer data, or production secrets in `ARENA_SECRET_TOKEN`. The full prompt and model response are broadcast to every connected presenter client.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Startup fails with a missing API key error | Set `GEMINI_API_KEY` before importing or starting Uvicorn. |
| Browser shows no live dashboard events | Confirm the presenter page uses the same host and port as the attendee page and that WebSockets are allowed through the network. |
| All attempts appear defended | Confirm the Gemini model is reachable and inspect the returned `reply`; only an exact token occurrence sets `hacked` to `true`. |
| Static files fail in a container | Run from the repository root when developing and ensure `static/` is copied into the image. |
| Slides look incomplete offline | The deck loads Tailwind, fonts, QR images, and confetti from external CDNs. Use an internet connection or vendor those assets for offline events. |

## Contributing

Keep changes focused, preserve the zero-build frontend approach, and test both attendee and presenter flows when changing the API or WebSocket payload. Do not commit credentials or live secret tokens.

## License

`MIT LICENSE` file is currently included in this repository.

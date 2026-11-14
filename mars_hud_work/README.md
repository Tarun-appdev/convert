# Mars Nav — deployable hackathon prototype

A deployable, browser-based MVP based on the supplied Mars Surface Navigation Build Specification. It includes a FastAPI service, responsive mission console, synthetic site resolver, A* route planner, simulated confidence levels, dust/SEP event injection, and a phone-friendly crew HUD over WebSockets.

## Important scope note

This is a **prototype MVP**, not the complete M0–M7 research/engineering system in the specification. It uses deterministic synthetic terrain and placeholder science sites. It does not ingest real Mars DEM/geology data, implement a validated EKF or real celestial/skyline/orbital solvers, or provide operational safety guarantees. Numeric outputs are illustrative only. Do not use it for real-world navigation or safety decisions.

## Run locally

Requires Python 3.11+.

```bash
python -m venv .venv
# macOS/Linux:
source .venv/bin/activate
# Windows PowerShell:
# .venv\\Scripts\\Activate.ps1
pip install -r requirements.txt
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

Open http://localhost:8000. API docs: http://localhost:8000/docs.

## Deploy on Render

1. Create a GitHub repository and push the contents of this folder.
2. In Render, choose **New → Blueprint** and connect the repository.
3. Render detects `render.yaml`; create the web service.
4. Open the generated `onrender.com` URL.

Alternative: deploy the Dockerfile to any container host. The service listens on `$PORT`.

## Phone HUD

For local testing, the server **must** listen on all network interfaces. Start it with `uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload` (not the default `127.0.0.1`). Connect the laptop and phone to the same Wi-Fi, find the laptop IPv4/LAN address (for example `192.168.1.25`), and on the phone open `http://192.168.1.25:8000`. Create a session on that address and open the displayed `http://192.168.1.25:8000/hud?code=XXXXXX` link. Allow Python/Uvicorn through the laptop firewall on private networks. Do not use `localhost` or `127.0.0.1` on the phone: those addresses point back to the phone itself. On a public HTTPS deployment, open the deployed server URL instead. The HUD also falls back to periodic HTTP telemetry polling if a mobile browser cannot keep a WebSocket open.

## Included API

- `GET /api/health`
- `GET /api/sites`
- `POST /api/resolve`
- `POST /api/plan`
- `POST /api/session`
- `POST /api/session/{code}/configure`
- `POST /api/session/{code}/start`
- `POST /api/session/{code}/pause`
- `POST /api/session/{code}/event` with `{"kind":"dust"}`, `{"kind":"sep"}`, or `{"kind":"clear"}`
- `GET /api/session/{code}/state`
- `WS /ws/{code}`

## Current implementation vs. specification

| Area | MVP status |
|---|---|
| Mission console | Implemented |
| Candidate science-site resolver | Rule-based synthetic baseline |
| Route planner | A* over synthetic cost field |
| High/medium/low confidence comparison | Simplified uncertainty-dependent route cost and buffers |
| Fix-stop insertion | Demonstration heuristic |
| Dust and SEP injection | Implemented as scripted HUD events |
| Phone HUD | Responsive PWA-style page + WebSocket stream |
| Real Mars datasets / GIS preprocessing | Not implemented |
| EKF and sensor fusion | Not implemented |
| True panorama / camera AR | Not implemented |
| Consumables and validated human model | Placeholder telemetry only |
| Cloud persistence / multi-user accounts | Not implemented |

## Configuration and data

The app intentionally labels the environment as synthetic. The coordinates are local demonstration coordinates, not verified Jezero survey coordinates. The external font import is decorative; the app itself does not require external API calls at runtime.

## Crew HUD mission data flow

The crew HUD at `/hud?code=JOINCODE` now reads the mission plan from the server and renders its waypoint polyline, waypoint sequence, next-waypoint bearing/range, heading, ground speed, localization confidence, ETA, dust, power, O₂ and alerts. It receives live frames over the existing WebSocket.

The mission console's **Create session** + **Generate route** workflow sends the route to the session. An external mission-design service can also upload a plan:

```http
POST /api/session/JOINCODE/mission
Content-Type: application/json
```

```json
{
  "mission": {
    "plan_id": "mission-001",
    "name": "Delta traverse",
    "mode": "walk",
    "eta_min": 42.5,
    "distance_m": 1850,
    "waypoints_enu_m": [[0, 0, 0], [120, 80, 0], [260, 140, 0]]
  }
}
```

Read the active plan with `GET /api/session/JOINCODE/mission`. Coordinates are `[east_m, north_m, elevation_m]` in the prototype's local frame. The HUD uses the same origin and coordinate convention as the route planner. This endpoint is an integration seam for the mission designer; add authentication and schema/version validation before exposing it publicly.

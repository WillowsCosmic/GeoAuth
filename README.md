# GeoAuth

A passwordless authentication system where users log in by clicking an **ordered sequence of points on a map** instead of typing a password. Raw coordinates are never stored — the click sequence is snapped to a tolerance grid, salted, and hashed with PBKDF2, exactly like a conventional password.

---

## How It Works

1. **Register** — Enter a username and click 3–6 points on the map in a specific order.
2. **Login** — Re-click the same points in the same order. Each point only needs to be "close enough" (within the tolerance grid cell, ~55m by default).
3. **Security** — The ordered sequence is hashed with PBKDF2-HMAC-SHA256 (310,000 iterations). A full database leak does not expose usable coordinates.

---

## Features

- **Grid snapping** — Clicks within the same tolerance cell hash identically, making human re-clicking reliable without sacrificing security.
- **Ordered sequence** — Point order matters. Clicking the same locations in a different order is a different credential.
- **PBKDF2 hashing** — Per-user salt + 310,000 iterations (OWASP 2023 recommendation).
- **Rate limiting** — Accounts are locked for 60 seconds after 5 consecutive failed attempts.
- **Username enumeration protection** — Wrong username and wrong points return an identical error response.
- **Entropy API** — `/api/entropy` calculates the effective keyspace of the current map view and compares it to an equivalent alphanumeric password length.

---

## Project Structure

```
GeoAuth/
├── app.py               # Flask backend
├── geodec.db            # SQLite database (auto-created on first run)
├── requirements.txt
├── static/
│   ├── app.js
│   └── style.css
└── templates/
    └── index.html
```

---

## Setup

**1. Clone the repository**
```bash
git clone https://github.com/WillowsCosmic/GeoAuth.git
cd GeoAuth
```

**2. Create and activate a virtual environment**
```bash
python -m venv geoenv
# Windows
geoenv\Scripts\activate
# macOS / Linux
source geoenv/bin/activate
```

**3. Install dependencies**
```bash
pip install -r requirements.txt
```

**4. Run the app**
```bash
python app.py
```

Open [http://localhost:5000](http://localhost:5000) in your browser.

---

## API Reference

### `POST /api/register`
Register a new user with a map-click sequence.

**Body (JSON)**
```json
{
  "username": "alice",
  "points": [
    { "lat": 51.505, "lng": -0.09 },
    { "lat": 48.858, "lng": 2.294 },
    { "lat": 40.712, "lng": -74.006 }
  ]
}
```

**Response**
```json
{ "ok": true, "message": "Registered. Remember your click order — it IS your password." }
```

---

## Screenshots
<img width="1920" height="942" alt="image" src="https://github.com/user-attachments/assets/861b7303-2e92-4f2b-9bd1-31fa2c561724" />
**Register**

<img width="1920" height="942" alt="image" src="https://github.com/user-attachments/assets/3f4ecd58-2b61-4c39-9a9e-9d4bd9b96d67" />
**Authentication**



### `POST /api/login`
Authenticate with a username and click sequence.

**Body (JSON)** — same format as `/api/register`

**Response**
```json
{ "ok": true, "message": "Authenticated." }
```

---

### `GET /api/entropy`
Calculate the keyspace and entropy for a given map viewport.

**Query Parameters**

| Parameter | Description |
|-----------|-------------|
| `south`   | Southern latitude bound |
| `north`   | Northern latitude bound |
| `west`    | Western longitude bound |
| `east`    | Eastern longitude bound |
| `points`  | Number of click points (default: 3) |

**Response**
```json
{
  "ok": true,
  "grid_cells_in_view": 12400,
  "keyspace": 1882214668800,
  "entropy_bits": 40.8,
  "equivalent_alnum_password_length": 6.9
}
```

---

## Configuration

Key constants in `app.py`:

| Constant | Default | Description |
|----------|---------|-------------|
| `GRID_SIZE_DEG` | `1.0` | Tolerance cell size in degrees (~55m at 0.0005°). Increase for looser matching, decrease to tighten. |
| `MIN_POINTS` | `3` | Minimum clicks required |
| `MAX_POINTS` | `6` | Maximum clicks allowed |
| `PBKDF2_ITERATIONS` | `310,000` | Hash iterations (OWASP 2023) |
| `MAX_FAILED_ATTEMPTS` | `5` | Failed logins before lockout |
| `LOCKOUT_SECONDS` | `60` | Lockout duration |

---

## Security Notes

- The tolerance grid is the primary security tuning knob. Coarser grid = smaller keyspace (see `/api/entropy`); finer grid = harder for users to reproduce their own clicks.
- Rate limiting is in-memory and resets on server restart. For production, use a persistent store (e.g. Redis).
- This project is intended for educational/demonstration purposes.

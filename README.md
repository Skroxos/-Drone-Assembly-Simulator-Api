# Drone Assembly Simulator — REST API

[![.NET](https://img.shields.io/badge/.NET-8.0-512BD4?style=for-the-badge&logo=dotnet)](https://dotnet.microsoft.com/)
[![Azure](https://img.shields.io/badge/Hosted_on-Azure_App_Service-0089D6?style=for-the-badge&logo=microsoft-azure)](https://azure.microsoft.com/)
[![Database](https://img.shields.io/badge/Database-Azure_MySQL-4479A1?style=for-the-badge&logo=mysql)](https://azure.microsoft.com/en-us/products/mysql/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](../Drone-Assembly-Simulator/blob/main/LICENSE)

Backend REST API for the [Drone Assembly Simulator](https://github.com/Skroxos/Drone-Assembly-Simulator) — a 3D Unity game where players assemble a drone as fast as possible. This API handles leaderboard score submission and retrieval.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | ASP.NET Core 8 Minimal API |
| Database | Azure MySQL Flexible Server via EF Core (Pomelo) |
| Security | SHA-256 signed request payloads |
| Rate Limiting | Built-in `System.Threading.RateLimiting` (Fixed Window, IP-based) |
| Hosting | Azure App Service (Linux) |

---

## API Endpoints

### `POST /api/assembly/save`
Submits a completed assembly session to the leaderboard.

**Request body:**
```json
{
  "name": "PlayerName",
  "finishTime": 42.37,
  "hash": "e3b0c44298fc1c149afb..."
}
```

| Field | Type | Description |
|---|---|---|
| `name` | `string` | Player's display name |
| `finishTime` | `float` | Assembly completion time in seconds |
| `hash` | `string` | SHA-256 signature for request validation |

**Responses:**
- `201 Created` — Score saved successfully, returns `{ "id": "..." }`
- `400 Bad Request` — Missing or invalid fields
- `401 Unauthorized` — Invalid SHA-256 hash
- `429 Too Many Requests` — Rate limit exceeded

---

### `GET /api/assembly/top10`
Returns the top 10 fastest assembly times.

**Response:**
```json
[
  { "name": "PlayerName", "finishTime": 42.37 },
  ...
]
```

**Responses:**
- `200 OK` — Array of top 10 sessions ordered by finish time (ascending)
- `429 Too Many Requests` — Rate limit exceeded

---

## Security

### SHA-256 Request Signing
Every POST request must include a `hash` field computed from the payload and a shared secret key. The server recomputes the hash and compares it using constant-time comparison to prevent timing attacks.

**Hash construction:**
```
SHA-256( name + finishTime + secretKey )
```

Where `finishTime` is formatted as a two-decimal float using `InvariantCulture` (e.g. `"42.37"`).

The secret key is stored in **Azure App Service Configuration** and never committed to source control.

### Rate Limiting
All endpoints are protected by an IP-based Fixed Window rate limiter:
- **5 requests** per **10 seconds** per IP address
- Exceeding the limit returns `429 Too Many Requests`

### SQL Injection Prevention
All database queries are handled through Entity Framework Core using parameterized queries, inherently protecting the database against SQL injection attacks.

---

## Running Locally

### Prerequisites
- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8)
- MySQL instance (or Docker)

### Setup

1. Clone the repository:
```bash
git clone https://github.com/Skroxos/Drone-Assembly-Simulator-Api.git
cd Drone-Assembly-Simulator-Api
```

2. Set your connection string and secret key via user secrets (never commit these):
```bash
cd DroneSimlator.Api
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=localhost;Database=drone_db;Uid=root;Pwd=yourpassword;"
dotnet user-secrets set "ApiSecretKey" "your-secret-key"
```

3. Apply database migrations:
```bash
dotnet ef database update
```

4. Run the API:
```bash
dotnet run
```

The API will be available at `https://localhost:5001`. Swagger UI is available at `/swagger` in the Development environment.

---

## Project Structure

```
DroneSimlator.Api/
├── Data/
│   └── SimulatorDbContext.cs      # EF Core DbContext
├── DTOs/
│   ├── SaveResultRequest.cs       # POST request model
│   └── LeaderboardResponse.cs     # GET response model
├── Models/
│   └── DroneAssemblySession.cs    # Database entity
├── Migrations/                    # EF Core migrations
└── Program.cs                     # Minimal API entry point & endpoints
```

---

## Related

- 🎮 [Drone Assembly Simulator](https://github.com/Skroxos/Drone-Assembly-Simulator) — The Unity game client that consumes this API

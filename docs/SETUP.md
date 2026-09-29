# SafeAR — Full Setup Guide

The short version lives in the [README quickstart](../README.md#quickstart--live-demo). This is the long version: every environment variable, the signing-key rules, and all three demo paths.

## Backend local setup

Do this once per clone. Every step is checked at boot, so skipping one fails
immediately and says which variable is missing rather than misbehaving later.

Commands are given for PowerShell and for bash/zsh where they differ. Run them
from the repository root.

### 1. Copy the environment template

```powershell
Copy-Item .env.example .env
```

```bash
cp .env.example .env
```

`.env` is gitignored. It holds every secret the backend has, and it never gets
committed, pasted into chat, or put in a screenshot.

### 2. Install dependencies

```bash
npm install
```

One install at the root covers all three workspaces — `backend/`, `frontend/`
and `dashboard/`.

### 3. Generate your development signing key

```bash
npm run keygen:dev
```

This writes a gitignored public key to
`backend/keys/cert-signing.dev.public.pem` and prints two lines to paste into
`.env`. It does not touch the shared team key.

> **Do not run `npm run keygen` for setup.** That one rotates the *team* signing
> key, and every certificate already issued stops verifying. It refuses to
> overwrite an existing key for exactly that reason. See
> [Team key vs dev key](#team-key-vs-dev-key) below.

### 4. Set `CERT_PRIVATE_KEY`

Paste the `CERT_PRIVATE_KEY=...` line that `keygen:dev` printed into `.env`,
replacing the `change_me_...` placeholder.

This is the only thing that can mint a certificate. It lives in `.env` and
nowhere else.

### 5. Set `ADMIN_API_KEY`

Pick your own value — there is no shared team admin key, and one must never be
committed. Any long random string works:

```powershell
[guid]::NewGuid().ToString()
```

```bash
openssl rand -hex 24
```

Leaving the placeholder logs a warning in development and is a hard boot failure
in production. Both are deliberate.

### 6. Verify `CERT_PUBLIC_KEY_PATH`

Set it to the public half of whichever key you are using:

| Working how | `CERT_PUBLIC_KEY_PATH` |
| --- | --- |
| Alone, with your own dev key | `./keys/cert-signing.dev.public.pem` |
| With the team's shared key | `./keys/cert-signing.public.pem` |

`keygen:dev` prints the first of these for you. The private key and this file
must be two halves of one pair; they are checked against each other at boot and
a mismatch fails loudly rather than producing certificates nobody can verify.

### 7. Create the demo data

```bash
npm run seed
```

This creates `backend/data/safear.db` and fills it with the demo workers,
modules and checkpoint manifest. Without it the database has no workers and
every attempt sync comes back `unknown_worker`.

The database file is local and gitignored. If you pull a branch with a newer
schema, the backend refuses to start and names the version it expected — delete
the file and re-seed:

```powershell
Remove-Item backend/data/safear.db
npm run seed
```

```bash
rm backend/data/safear.db
npm run seed
```

### 8. Start the backend

```bash
npm run dev:backend
```

### 9. Health check

```powershell
Invoke-RestMethod http://localhost:3000/api/health
```

```bash
curl http://localhost:3000/api/health
```

A healthy backend answers:

```json
{ "ok": true, "db": "up", "ts": "...", "requestId": "..." }
```

If it does not start, the error names the missing or invalid variable. Secrets
are never printed.

### Team key vs dev key

The two halves of an Ed25519 signing key are handled differently, and it
matters:

- The **private key** (`CERT_PRIVATE_KEY`) is a secret. It lives only in `.env`.
  Never commit it, never share it, never paste it into a chat.
- The **public key** is not a secret and is meant to be distributed.
  `backend/keys/cert-signing.public.pem` is committed on purpose so every machine
  verifies against the same issuer.

A **dev key** (`npm run keygen:dev`) is yours alone. Certificates you sign with
it verify on your machine and nowhere else, which is the point — a development
key must not be able to mint something the team would trust. It is enough for
building and testing the whole flow end to end on one laptop.

The **team key** (`npm run keygen`) is the shared issuer. For a demo spanning two
machines, one person generates it, commits the public half, and passes the
`CERT_PRIVATE_KEY` line to the others out of band. If each teammate generates
their own instead, a certificate issued on one laptop fails verification on
another with `bad_signature`.

Development signing keys and any production key are separate. Nothing generated
by `keygen:dev` should ever reach a real deployment.

## How the app finds the backend

The frontend resolves the backend address in this order, first match winning:

| Source | Use it for |
| --- | --- |
| `?api=http://host:3000` in the URL | A one-off override; it is remembered afterwards |
| Previously remembered value | Reloads after the override above |
| `window.SAFEAR_API_BASE` in `frontend/config.js` | APK builds |
| Frontend on port 5173 | Ordinary local development — nothing to configure |
| Same origin | A deployment where the backend serves the frontend |

Only `http` and `https` addresses are accepted, and a trailing slash is trimmed.

If no backend is reachable, nothing is lost. Attempts stay queued, certificates
stay pending, and both are sent the next time the app finds the server.

---

## Demo path 1 — browser on this machine

The everyday case, and the one that needs no configuration.

Three servers, three terminals:

```bash
npm run dev:backend
```

```bash
npm run dev:frontend
```

```bash
npm run dev:dashboard
```

Open `http://localhost:5173` for the training app. Served from port 5173, the
frontend calls port 3000 on the same host automatically — there is nothing to set.

`http://localhost:5174` redirects to the `/mobile/` Training Portal demo, **not**
the admin compliance dashboard — open `http://localhost:5174/admin.html` for that.

## Demo path 2 — phone browser over Wi-Fi

Good for checking layout, translations, sync and certificates on a real handset.
**It cannot demo AR** — see the limitation below.

Find your machine's LAN address (`ipconfig` on Windows, `ifconfig` or `ip addr`
elsewhere), then on the phone open:

```
http://192.168.1.50:5173/?api=http://192.168.1.50:3000
```

`192.168.1.50` is an RFC1918 example — substitute your own address. The `?api=`
value is remembered, so later loads do not need it.

The backend must be told to accept that origin. Append it to the **existing**
`ALLOWED_ORIGINS` line in `.env` — the environment value replaces the built-in
list rather than adding to it, so keep every entry that is already there:

```
ALLOWED_ORIGINS=http://localhost:5173,http://localhost:5174,http://localhost,https://localhost,capacitor://localhost,http://192.168.1.50:5173
```

Restart the backend afterwards. Both dev servers already listen on every network
interface, so no extra flag is needed, though a desktop firewall may ask you to
allow ports 3000 and 5173 the first time.

> **Limitation: no camera, so no AR.** Browsers only grant camera access on a
> secure origin, and a plain-HTTP LAN address is not one. `localhost` is trusted,
> a LAN IP is not. The app detects this and shows its unsupported-device view
> instead of failing messily, and everything that is not AR still works. **To demo
> AR on a phone, build the APK** — path 3, where the WebView origin is
> `http://localhost` and the camera is available.

## Demo path 3 — phone, as an APK

Meant to be the full demo, AR included. **Untested:** no APK has been built
yet, and AR will not get the camera until `android.permission.CAMERA` is added to
the generated `android/app/src/main/AndroidManifest.xml` (see
[`android/README.md`](../android/README.md)).

First, point the app at your backend. An installed app has no backend of its own,
and `localhost` on the phone means the phone. Edit `frontend/config.js`:

```js
window.SAFEAR_API_BASE = "http://192.168.1.50:3000";
```

Again an RFC1918 example — use your own address, and **do not commit it**. The
file ships with an empty default for exactly that reason. It holds an address,
not a credential; no key or token belongs in it.

Then build, from the project root:

```bash
npx cap add android
npx cap sync android
npx cap open android
```

`cap add android` generates the Android project and runs once per clone;
`cap sync android` copies `frontend/` into it and must run again after every
frontend change. The generated project is not committed — see `android/README.md`.

Two things are already configured for you:

- **CORS.** The Android WebView's origin is `http://localhost`, which is in the
  default `ALLOWED_ORIGINS`. Unlike path 2, no `.env` change is needed.
- **Cleartext HTTP.** Android has blocked plain HTTP by default since API 28, so
  the app could not reach `http://192.168.1.50:3000` at all. `capacitor.config.json`
  sets `server.cleartext: true` to permit it.

`cleartext` is a **demo-only setting**. It allows unencrypted traffic app-wide,
which is fine for a laptop backend on a closed Wi-Fi network and wrong for
anything real. A production build would serve the API over HTTPS and remove it.


# Packaging and deployment

## Even Hub CLI (`@evenrealities/evenhub-cli`)

The official CLI for Even Hub development and app management. Provides QR code generation, app packaging, and account login.

NPM: <https://www.npmjs.com/package/@evenrealities/evenhub-cli>

```bash
npm install -D @evenrealities/evenhub-cli
```

The package installs two binaries: `evenhub` and `eh` (alias). When installed locally (not globally), use via `npx evenhub` or npm scripts.

### CLI commands

| Command | Description |
|---|---|
| `evenhub login` | Log in to your Even Hub developer account |
| `evenhub init` | Generate a starter `app.json` manifest in the current directory |
| `evenhub qr [options]` | Generate a QR code for sideloading via the Even App |
| `evenhub pack <json> <project>` | Package a built app into a `.ehpk` file for distribution |

### `evenhub qr`

Generates a QR code to scan with the Even App on your phone, which loads your dev server URL on the glasses.

```bash
# Specify full URL directly (recommended)
npx evenhub qr --url "http://192.168.0.138:5173"

# Or build from parts – prompts for IP on first run, caches it
npx evenhub qr --http --port 5173

# With explicit IP (no prompt)
npx evenhub qr --http --ip 192.168.0.138 --port 5173

# Reset cached settings
npx evenhub qr --clear
```

| Option | Description |
|---|---|
| `--url <url>` | Full URL to encode (ignores other options) |
| `--ip <ip>` | IP address or hostname |
| `--port <port>` | Port number |
| `--http` / `--https` | Scheme (default: HTTPS) |
| `--path <path>` | URL path |
| `--external` | Open QR in external program instead of terminal |
| `--clear` | Clear cached scheme, IP, port and path |

**Tip:** Use your machine's local network IP (e.g. `192.168.x.x`), not `localhost` or `0.0.0.0` – the phone needs to reach your dev server over the network. Vite prints the network URL when starting with `--host 0.0.0.0`.

### `evenhub pack`

Packages a built app into a `.ehpk` file for distribution via the Even Hub portal.

```bash
# Build first, then pack
npx vite build
npx evenhub pack app.json dist -o myapp.ehpk

# Or combine in an npm script
# "pack": "npm run build && evenhub pack app.json dist -o myapp.ehpk"
```

| Option | Description |
|---|---|
| `<json>` | Path to `app.json` manifest (required) |
| `<project>` | Path to built output folder, e.g. `dist` (required) |
| `-o, --output <name>` | Output filename (default: `out.ehpk`) |
| `--no-ignore` | Include hidden files (starting with `.`) |
| `-c, --check` | Check if the package ID is available on Even Hub |

### `evenhub login`

Authenticates with your Even Hub developer account (required for publishing).

```bash
npx evenhub login
# Prompts for email and password
```

### `evenhub init`

Generates a starter `app.json` manifest file.

```bash
npx evenhub init
# Creates ./app.json with template values
```

## `app.json` manifest

Every Even Hub app needs an `app.json` at the project root. This describes the app for packaging and the Even Hub portal.

```json
{
  "package_id": "com.example.myapp",
  "edition": "202601",
  "name": "My app",
  "version": "1.0.0",
  "min_app_version": "0.1.0",
  "tagline": "Short description shown in Even Hub",
  "description": "Longer description of what the app does",
  "author": "Your Name",
  "entrypoint": "index.html",
  "permissions": {
    "network": ["api.example.com"],
    "fs": ["./assets"]
  }
}
```

**`package_id` rules:** Must be a valid reverse-domain name where each segment starts with a lowercase letter and contains only lowercase letters or numbers. No hyphens. Example: `com.myname.myapp` (valid), `com.my-name.my-app` (invalid).

**`permissions.network`:** List domains your app needs to access. Use `["*"]` for unrestricted network access (e.g. apps that connect to user-configured servers).

## Recommended npm scripts

```json
{
  "scripts": {
    "dev": "vite --host 0.0.0.0 --port 5173",
    "build": "vite build",
    "qr": "evenhub qr --http --port 5173",
    "pack": "npm run build && evenhub pack app.json dist -o myapp.ehpk"
  }
}
```

## Development workflow

1. Start the dev server: `npm run dev` (keep running in terminal 1)
2. Generate QR code: `npm run qr` (terminal 2 – or use `npx evenhub qr --url "http://<your-ip>:5173"`)
3. Scan the QR code with the Even App on your phone
4. The app loads on your glasses – Vite hot reload works for code changes

## Production packaging

1. `npm run pack` – builds and creates a `.ehpk` file

The `.ehpk` format is for future submission to the Even Hub portal (not yet available). Add `*.ehpk` to `.gitignore`.

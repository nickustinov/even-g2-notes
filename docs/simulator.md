# Simulator environment

[even-dev](https://github.com/BxNxM/even-dev) is a shared development environment for building and testing G2 apps with the Even Hub Simulator. It handles app discovery, dependency installation, Vite dev server configuration, and simulator launch.

Repository: <https://github.com/BxNxM/even-dev>

## Running apps

Install and run:

```bash
git clone https://github.com/BxNxM/even-dev.git
cd even-dev
npm install
./start-even.sh
```

The launcher lists all available apps and prompts you to pick one. You can also select directly:

```bash
APP_NAME=demo ./start-even.sh
```

To run a local app by path without any configuration:

```bash
APP_PATH=../my-app ./start-even.sh
```

External apps can be registered in `apps.json` at the even-dev root (git URLs are auto-cloned on first run):

```json
{
  "chess": "https://github.com/dmyster145/EvenChess",
  "my-local-app": "../my-local-app"
}
```

## App structure

A G2 app is a regular web app. There is no special framework or required interface – just HTML, TypeScript, and the Even Hub SDK.

Minimal file structure:

```
my-app/
  index.html          <- entry point
  package.json        <- dependencies and scripts
  vite.config.ts      <- dev server config
  app.json            <- app metadata (for packaging)
  src/
    main.ts           <- app bootstrap
    styles.css        <- stylesheet
```

**`index.html`** – a standard HTML page that loads your app:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>My App</title>
  <link rel="stylesheet" href="/src/styles.css">
</head>
<body>
  <div id="app"></div>
  <script type="module" src="/src/main.ts"></script>
</body>
</html>
```

**`package.json`** – at minimum, the Even Hub SDK and Vite:

```json
{
  "name": "my-even-app",
  "scripts": {
    "dev": "vite --host 0.0.0.0 --port 5173",
    "build": "vite build"
  },
  "dependencies": {
    "@evenrealities/even_hub_sdk": "^0.0.7"
  },
  "devDependencies": {
    "typescript": "^5.9.3",
    "vite": "^7.3.1"
  }
}
```

**`vite.config.ts`** – keep it minimal:

```typescript
import { defineConfig } from 'vite'

export default defineConfig({
  server: {
    host: true,
    port: 5173,
  },
})
```

**`src/styles.css`** – import the even-realities-ui stylesheet for consistent browser UI, then add your own styles:

```css
@import "../node_modules/@jappyjan/even-realities-ui/dist/even-realities-ui.css";

body {
  font-family: system-ui, -apple-system, sans-serif;
  max-width: 480px;
  margin: 0 auto;
  padding: 16px;
}
```

**Tips:**

- **Use inline styles or plain CSS** – avoid CSS frameworks like Tailwind that require build plugins. Your app may be served through even-dev's Vite config, which won't have those plugins.
- **Keep it standalone** – your app should work with just `npm run dev`. Don't depend on even-dev's infrastructure.
- **If your app needs a backend server**, put it in a `server/` directory with its own `package.json`. Even-dev auto-detects and starts it alongside Vite.
- **Use `@jappyjan/even-realities-ui`** for the browser settings page – Card, Button, Input, Text components give a consistent look across apps. See [browser UI](browser-ui.md).

**Reference apps:**

| App | Description | Source |
|-----|-------------|--------|
| [chess](https://github.com/dmyster145/EvenChess) | Full app with tests, linting, modular architecture | Complex reference |
| [reddit](https://github.com/fuutott/rdt-even-g2-rddit-client) | Clean app with `app.json` packaging, API proxy, evenhub-cli integration | Packaging reference |
| [weather](https://github.com/nickustinov/weather-even-g2) | Settings UI with even-realities-ui components, plain CSS | Simple reference |

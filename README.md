# The Ultimate MERN Stack Project Setup Guide

**Last Updated:** February 2026  
**Architecture:** Monorepo · ES Modules · Express 5 · Vite · Docker

A clean, production-ready MERN skeleton. Every file is provided. Every decision is explained. Follow it top to bottom and you'll have a running app in under 15 minutes.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Phase 1 — Monorepo Foundation](#phase-1--monorepo-foundation)
3. [Phase 2 — Backend (Express 5 API)](#phase-2--backend-express-5-api)
4. [Phase 3 — Frontend (Vite + React)](#phase-3--frontend-vite--react)
5. [Phase 4 — Orchestration](#phase-4--orchestration)
6. [Phase 5 — Database (Docker)](#phase-5--database-docker)
7. [Phase 6 — AI Onboarding (Claude Code)](#phase-6--ai-onboarding-claude-code)
8. [Running the App](#running-the-app)
9. [What to Build Next](#what-to-build-next)
10. [Toolkit Reference](#toolkit-reference)

---

## Prerequisites

| Tool | Version | Why |
|---|---|---|
| **Node.js** | v22 LTS (or v24 LTS) | Even-numbered releases = Long Term Support. Express 5 requires ≥ v18. |
| **npm** | Ships with Node | Package manager. No extra install needed. |
| **Git** | Any recent | Version control. |
| **Docker Desktop** | Latest | Runs MongoDB in a container so you never install it locally. |

> **Optional:** [Claude Code](https://docs.claude.com) (`npm install -g @anthropic-ai/claude-code`) — an AI coding assistant that lives in your terminal. Phase 6 covers setup.

Verify your environment before continuing:

```bash
node -v   # Should print v22.x.x or v24.x.x
npm -v    # Should print 10.x+
git -v    # Any version
docker -v # Any version
```

---

## Phase 1 — Monorepo Foundation

A monorepo means one Git repository holds both client and server. The root `package.json` exists solely for orchestration — it never holds application code.

### 1.1 Initialize the project

```bash
mkdir my-mern-app
cd my-mern-app
git init
npm init -y
```

### 1.2 Create `.gitignore` (do this immediately)

This is the single most important file you create first. One leaked `.env` with a database URI or API key can compromise your entire project.

```
# Dependencies
node_modules/

# Environment secrets
.env
.env.local
.env.*.local

# Build output
dist/
build/

# OS / Editor junk
.DS_Store
Thumbs.db
.vscode/
.idea/

# Logs
*.log
npm-debug.log*
```

> **Why no `.env.example` in `.gitignore`?** Because `.env.example` is meant to be committed — it documents which environment variables the project needs, without containing actual secrets.

### 1.3 Install the orchestration tool

```bash
npm install concurrently --save-dev
```

**What this does:** `concurrently` runs multiple shell commands in parallel. We use it to start the client and server with a single `npm run dev` from the root.

---

## Phase 2 — Backend (Express 5 API)

We build a secure, structured API using Express 5 and ES Modules.

### Why Express 5?

Express 5 is now the default on npm (as of March 2025). Key improvements over v4:
- **Native async error handling** — rejected promises are automatically caught and forwarded to your error middleware. No more `try/catch` in every route or `express-async-handler` wrappers.
- **Improved security** — route matching uses a stricter path parser that prevents ReDoS (Regular Expression Denial of Service) attacks.
- **Requires Node.js 18+** — which we already have.

### 2.1 Setup and dependencies

```bash
mkdir server && cd server
npm init -y
```

Install production dependencies:

```bash
npm install express mongoose dotenv cors helmet morgan
```

Install dev dependencies:

```bash
npm install --save-dev nodemon
```

| Package | Purpose |
|---|---|
| `express` | Web framework. Handles routing, middleware, HTTP. |
| `mongoose` | ODM (Object Data Modeling) for MongoDB. Provides schemas, validation, and query building. |
| `dotenv` | Loads variables from `.env` into `process.env` so secrets never touch your code. |
| `cors` | Controls which origins (domains) can call your API. Defaults to "allow all" in dev. |
| `helmet` | Sets secure HTTP headers automatically (prevents clickjacking, XSS sniffing, etc). |
| `morgan` | Logs every HTTP request to the console (`GET /api/users 200 12ms`). Invaluable for debugging. |
| `nodemon` | Watches your files and restarts the server on save. Dev only. |

### 2.2 Enable ES Modules

Open `server/package.json` and add `"type": "module"`. This tells Node.js to use `import`/`export` instead of the older `require`/`module.exports` syntax.

```json
{
  "name": "server",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  }
}
```

> **Why ES Modules?** It's the official JavaScript standard. Your frontend (Vite/React) already uses it. Using ESM everywhere means consistent syntax across your entire codebase, and it enables tree-shaking (dead code elimination) in tools that support it.

### 2.3 Folder structure

Create this inside `/server`:

```
server/
├── config/
│   └── db.js              # Database connection
├── controllers/
│   └── exampleController.js
├── middleware/
│   └── errorMiddleware.js  # Centralized error handling
├── models/
│   └── ExampleModel.js
├── routes/
│   └── exampleRoutes.js
├── .env                    # Your secrets (never committed)
├── .env.example            # Template (committed)
├── package.json
└── server.js               # Entry point
```

This is the **MVC pattern** (Model-View-Controller), adapted for APIs where the "View" is JSON. Each layer has one job: models define data shape, controllers hold business logic, routes map URLs to controllers.

Create all the folders now:

```bash
mkdir config controllers middleware models routes
```

### 2.4 Environment files

Create `server/.env`:

```
NODE_ENV=development
PORT=5000
MONGO_URI=mongodb://localhost:27017/my-mern-app
```

Create `server/.env.example` (this gets committed to Git as documentation):

```
NODE_ENV=development
PORT=5000
MONGO_URI=mongodb://localhost:27017/my-mern-app
```

> **Tip:** When you move to a cloud database (MongoDB Atlas), you replace `MONGO_URI` in `.env` with your Atlas connection string. The code doesn't change.

### 2.5 Database connection — `server/config/db.js`

```javascript
import mongoose from 'mongoose';

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGO_URI);
    console.log(`✅ MongoDB connected: ${conn.connection.host}`);
  } catch (error) {
    console.error(`❌ MongoDB connection error: ${error.message}`);
    process.exit(1);
  }
};

export default connectDB;
```

**What's happening:** Mongoose connects to MongoDB using the URI from your `.env`. If it fails (wrong URI, database not running), the process exits with code 1 so you see the error immediately instead of the app silently hanging.

> **Note:** Mongoose 8+ no longer requires options like `useNewUrlParser` or `useUnifiedTopology`. They are now defaults. If you see them in old tutorials, skip them.

### 2.6 Error middleware — `server/middleware/errorMiddleware.js`

```javascript
// Catches requests to routes that don't exist
export const notFound = (req, res, next) => {
  const error = new Error(`Not Found - ${req.originalUrl}`);
  res.status(404);
  next(error);
};

// Catches all errors thrown anywhere in the app
export const errorHandler = (err, req, res, next) => {
  const statusCode = res.statusCode === 200 ? 500 : res.statusCode;

  res.status(statusCode).json({
    message: err.message,
    stack: process.env.NODE_ENV === 'production' ? null : err.stack,
  });
};
```

**Why centralized errors?** Without this, every controller would need its own error formatting. This middleware gives you consistent JSON error responses and hides stack traces from users in production.

### 2.7 Example model — `server/models/ExampleModel.js`

```javascript
import mongoose from 'mongoose';

const exampleSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: [true, 'Name is required'],
      trim: true,
    },
    description: {
      type: String,
      default: '',
    },
  },
  {
    timestamps: true, // Adds createdAt and updatedAt automatically
  }
);

const Example = mongoose.model('Example', exampleSchema);

export default Example;
```

### 2.8 Example controller — `server/controllers/exampleController.js`

```javascript
import Example from '../models/ExampleModel.js';

// @desc    Get all examples
// @route   GET /api/examples
export const getExamples = async (req, res) => {
  const examples = await Example.find({});
  res.json(examples);
};

// @desc    Create an example
// @route   POST /api/examples
export const createExample = async (req, res) => {
  const { name, description } = req.body;

  if (!name) {
    res.status(400);
    throw new Error('Name field is required');
  }

  const example = await Example.create({ name, description });
  res.status(201).json(example);
};
```

> **No `try/catch` needed.** Express 5 automatically catches rejected promises from `async` route handlers and forwards them to the error middleware. This is the biggest quality-of-life improvement over Express 4, where you'd need `express-async-handler` or manual `try/catch` blocks everywhere.

### 2.9 Example routes — `server/routes/exampleRoutes.js`

```javascript
import { Router } from 'express';
import { getExamples, createExample } from '../controllers/exampleController.js';

const router = Router();

router.route('/').get(getExamples).post(createExample);

export default router;
```

### 2.10 Entry point — `server/server.js`

```javascript
import express from 'express';
import dotenv from 'dotenv';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';
import connectDB from './config/db.js';
import exampleRoutes from './routes/exampleRoutes.js';
import { notFound, errorHandler } from './middleware/errorMiddleware.js';

// Load environment variables BEFORE anything else
dotenv.config();

// Connect to database
connectDB();

const app = express();
const PORT = process.env.PORT || 5000;

// --- Middleware ---

// Security: sets headers like X-Content-Type-Options, Strict-Transport-Security, etc.
app.use(helmet());

// CORS: in development, allow all origins. In production, lock this down.
app.use(cors());

// Logging: prints "GET /api/examples 200 4.231 ms" to your terminal
app.use(morgan('dev'));

// Body parsing: allows req.body to contain JSON data from POST/PUT requests
app.use(express.json());

// --- Routes ---
app.get('/', (req, res) => {
  res.json({ message: 'API is running' });
});

app.use('/api/examples', exampleRoutes);

// --- Error handling (must be registered LAST) ---
app.use(notFound);
app.use(errorHandler);

// --- Start server with graceful shutdown ---
const server = app.listen(PORT, () => {
  console.log(`🚀 Server running in ${process.env.NODE_ENV} mode on port ${PORT}`);
});

// Graceful shutdown: close connections cleanly when process is killed
process.on('SIGTERM', () => {
  console.log('SIGTERM received. Shutting down gracefully...');
  server.close(() => process.exit(0));
});
```

**Key details:**
- `dotenv.config()` is called before `connectDB()` so the database URI is available.
- `express.json()` replaces the old `body-parser` package (it's been built into Express since v4.16).
- Error middleware is registered last because Express processes middleware in order — errors bubble down to it.
- Graceful shutdown ensures open database connections and in-flight requests finish before the process exits (important for Docker and cloud deployments).

Now go back to the root:

```bash
cd ..
```

---

## Phase 3 — Frontend (Vite + React)

### Why Vite?

Vite replaces Create React App (CRA), which is no longer maintained. Vite is faster (uses native ES Modules in dev — no bundling step), supports Hot Module Replacement (instant updates without page reload), and has a smaller config surface.

### 3.1 Scaffold the React app

From the **root** folder:

```bash
npm create vite@latest client -- --template react
cd client
npm install
```

### 3.2 Remove the nested `.git`

Vite doesn't create a `.git` anymore in recent versions, but if one exists, remove it to avoid Git submodule issues:

```bash
rm -rf .git
```

### 3.3 Configure the API proxy — `client/vite.config.js`

This is the "secret sauce" that eliminates CORS issues during development:

```javascript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:5000',
        changeOrigin: true,
        secure: false,
      },
    },
  },
});
```

**How the proxy works:** When your React code calls `fetch('/api/examples')`, Vite intercepts requests starting with `/api` and forwards them to `http://localhost:5000/api/examples`. The browser thinks it's talking to the same origin (port 5173), so there's no CORS error. In production, you'd serve the built React files from Express directly or use a reverse proxy like Nginx.

> **Rule:** Never hardcode `http://localhost:5000` in your React components. Always use relative paths like `/api/examples`. The proxy handles the rest.

### 3.4 Clean up starter files (optional)

Remove the default Vite boilerplate you won't need:

```bash
rm src/App.css src/assets/react.svg
```

Replace `client/src/App.jsx` with a minimal component that tests the API:

```jsx
import { useState, useEffect } from 'react';

function App() {
  const [message, setMessage] = useState('Loading...');

  useEffect(() => {
    fetch('/api/examples')
      .then((res) => res.json())
      .then((data) => setMessage(JSON.stringify(data)))
      .catch(() => setMessage('API not reachable'));
  }, []);

  return (
    <div>
      <h1>MERN Stack</h1>
      <p>API Response: {message}</p>
    </div>
  );
}

export default App;
```

Go back to the root:

```bash
cd ..
```

---

## Phase 4 — Orchestration

### 4.1 Wire everything together

Open the **root** `package.json` and replace the `"scripts"` section:

```json
{
  "scripts": {
    "install:all": "npm install && npm install --prefix client && npm install --prefix server",
    "server": "npm run dev --prefix server",
    "client": "npm run dev --prefix client",
    "dev": "concurrently \"npm run server\" \"npm run client\""
  }
}
```

| Script | What it does |
|---|---|
| `npm run install:all` | Installs dependencies for root, client, and server in one command. |
| `npm run server` | Starts only the backend (with nodemon). |
| `npm run client` | Starts only the frontend (with Vite). |
| `npm run dev` | Starts **both** simultaneously. This is your daily driver. |

### 4.2 First-time setup for collaborators

When someone clones your repo, they run:

```bash
npm run install:all
```

Then they're ready to develop.

---

## Phase 5 — Database (Docker)

### Why Docker for MongoDB?

Installing MongoDB locally differs by operating system, version, and architecture. Docker guarantees the same behavior everywhere. One command starts it; one command stops it.

### 5.1 Create `docker-compose.yml` in the root

```yaml
services:
  mongo:
    image: mongo:8
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db

volumes:
  mongo-data:
```

> **Note:** The `version` key (e.g., `version: '3.8'`) is deprecated in Docker Compose V2 and is now ignored. Omit it.

### 5.2 Start the database

```bash
docker compose up -d
```

The `-d` flag runs it in the background. Your data persists in the `mongo-data` volume even if you stop the container.

Useful commands:

```bash
docker compose down      # Stop the container
docker compose logs mongo # View database logs
```

> **Alternative:** If you don't want Docker, install MongoDB Community Edition locally or use a free [MongoDB Atlas](https://www.mongodb.com/atlas) cluster. Just update `MONGO_URI` in `server/.env`.

---

## Phase 6 — AI Onboarding (Claude Code)

> **This phase is optional.** Skip it if you don't use Claude Code.

We initialize the AI assistant *after* the project structure is in place. This way, Claude scans your directory and understands the monorepo layout, file conventions, and module system from the start.

### 6.1 Initialize

From the **root** directory:

```bash
claude init
```

### 6.2 Create `CLAUDE.md`

This file acts as persistent instructions for Claude Code — like a `.editorconfig` but for your AI assistant. Create `CLAUDE.md` in the root:

```markdown
# Project Rules — MERN Monorepo

## Architecture
- Monorepo: root orchestrates, `/client` is Vite/React, `/server` is Express 5/Node.js.
- ES Modules everywhere. Use `import`/`export`. Never use `require`.
- Database: Mongoose ODM. Connection logic in `/server/config/db.js`.

## Running
- `npm run dev` from the ROOT starts both client and server.
- Client: port 5173. Server: port 5000.

## API Calls from React
- The client uses a Vite proxy: `/api` → `http://localhost:5000`.
- Always use relative paths: `fetch('/api/users')`.
- NEVER hardcode `http://localhost:5000` in React code.

## Error Handling
- Express 5 catches async errors automatically. Do NOT wrap controllers in try/catch or use express-async-handler.
- Throw errors with `res.status(400); throw new Error('message')`.

## Conventions
- Models: PascalCase (`UserModel.js`), schemas use `timestamps: true`.
- Controllers: camelCase exports (`getUsers`, `createUser`).
- Routes: mounted at `/api/<resource>`.
- Secrets: never commit `.env`. Use `.env.example` for documentation.
```

---

## Running the App

### First time

```bash
# 1. Start the database
docker compose up -d

# 2. Install everything
npm run install:all

# 3. Start developing
npm run dev
```

### Every day after that

```bash
docker compose up -d   # if not already running
npm run dev
```

Your app is live:

| Service | URL |
|---|---|
| Frontend | http://localhost:5173 |
| Backend  | http://localhost:5000 |
| API Test | http://localhost:5000/api/examples |

---

## What to Build Next

This skeleton is deliberately minimal. Here's what to add based on your project:

- **Authentication** — JWT with `jsonwebtoken` and `bcryptjs`, or session-based auth with `express-session`.
- **Validation** — Use `express-validator` or Zod on incoming request data.
- **File uploads** — `multer` for handling multipart form data.
- **Testing** — `vitest` for frontend, `supertest` + Node's built-in test runner for backend.
- **Rate limiting** — `express-rate-limit` to protect against abuse.
- **Production build** — Serve the Vite build from Express:
  ```javascript
  // In server.js, after routes:
  if (process.env.NODE_ENV === 'production') {
    app.use(express.static('../client/dist'));
    app.get('*', (req, res) => res.sendFile(resolve('../client/dist/index.html')));
  }
  ```

---

## Toolkit Reference

Quick reference for every tool in this stack and why it was chosen.

### Express 5

The web framework. Handles routing, middleware, and HTTP request/response. Version 5 (now the npm default) adds native async error handling — rejected promises in route handlers are automatically passed to error middleware. This eliminates the need for `express-async-handler` or manual `try/catch` blocks that were required in Express 4.

### Mongoose

The Object Data Modeling library for MongoDB. Instead of writing raw MongoDB queries, you define schemas with types, validation, and defaults. Mongoose gives you `User.find()`, `User.create()`, and `User.findByIdAndUpdate()` instead of `db.collection('users').insertOne()`.

### Helmet

A single `app.use(helmet())` sets 15+ security-related HTTP headers. It prevents attacks like clickjacking (`X-Frame-Options`), MIME type sniffing (`X-Content-Type-Options`), and helps enforce HTTPS (`Strict-Transport-Security`). Zero configuration needed.

### Morgan

HTTP request logger. When something goes wrong, Morgan's output tells you exactly what request was made, what status code was returned, and how long it took. The `'dev'` format gives colorized, concise output like `GET /api/users 200 4.231 ms`.

### CORS

Controls which domains can call your API. In development, `app.use(cors())` allows all origins. In production, you'd restrict it:
```javascript
app.use(cors({ origin: 'https://yourdomain.com' }));
```

### Dotenv

Reads a `.env` file and loads its key-value pairs into `process.env`. This means your code references `process.env.MONGO_URI` and the actual value stays out of your source code and Git history.

### Vite

Replaces Create React App (which is no longer maintained). Vite serves files via native ES Modules during development, making startup and hot reloads near-instant. The proxy configuration eliminates CORS issues by forwarding `/api` requests to your Express server.

### Nodemon

Watches your server files and restarts Node.js automatically when you save. Dev-only tool. In production, you use `node server.js` directly (or a process manager like PM2).

### Concurrently

Runs multiple npm scripts in parallel. Our `npm run dev` uses it to start both the Vite dev server and the Express server in one terminal.

### Docker Compose

Defines and runs the MongoDB container. One `docker compose up -d` gives you a database. The volume mount (`mongo-data`) ensures your data survives container restarts. No OS-specific MongoDB installation needed.

---

## Final Project Structure

```
my-mern-app/
├── client/                     # Vite + React frontend
│   ├── public/
│   ├── src/
│   │   ├── App.jsx
│   │   └── main.jsx
│   ├── index.html
│   ├── package.json
│   └── vite.config.js
├── server/                     # Express 5 backend
│   ├── config/
│   │   └── db.js
│   ├── controllers/
│   │   └── exampleController.js
│   ├── middleware/
│   │   └── errorMiddleware.js
│   ├── models/
│   │   └── ExampleModel.js
│   ├── routes/
│   │   └── exampleRoutes.js
│   ├── .env
│   ├── .env.example
│   ├── package.json
│   └── server.js
├── .gitignore
├── CLAUDE.md                   # AI assistant rules (optional)
├── docker-compose.yml
├── package.json                # Root orchestration only
└── README.md
```

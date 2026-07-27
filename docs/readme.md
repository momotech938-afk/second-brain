# Second Brain

A personal knowledge app: a place to capture, organize and find notes and ideas.

> Status: early setup. The client and server are fresh scaffolds with no features yet.

## Tech stack

| Part | Folder | Stack |
|------|--------|-------|
| Client | [`client/`](../client) | Angular 22, Tailwind CSS 4, Vitest |
| Server | [`server/`](../server) | NestJS 12 (Express), TypeScript, Vitest, oxlint |

## Project structure

```
second-brain/
├── client/   # Angular web app
├── server/   # NestJS API
└── docs/     # Project docs
```

## Getting started

Requirements: Node.js 22+ and npm.

### Server

```bash
cd server
npm install
npm run start:dev      # http://localhost:3000 (set PORT to change)
```

### Client

```bash
cd client
npm install
npm start              # http://localhost:4200
```

Run both in separate terminals during development.

## Scripts

### Server (`server/`)

| Command | What it does |
|---------|--------------|
| `npm run start:dev` | Start with hot reload |
| `npm run build` | Build to `dist/` |
| `npm run start:prod` | Run the built app |
| `npm test` | Unit tests |
| `npm run test:e2e` | End-to-end tests |
| `npm run test:cov` | Tests with coverage |
| `npm run lint` | Lint with oxlint |
| `npm run format` | Format with Prettier |

### Client (`client/`)

| Command | What it does |
|---------|--------------|
| `npm start` | Dev server with live reload |
| `npm run build` | Production build |
| `npm run watch` | Rebuild on change (dev config) |
| `npm test` | Unit tests |

## License

Private project, not licensed for reuse.

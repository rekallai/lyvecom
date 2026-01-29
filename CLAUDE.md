# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

**Lyvecom** is a payment processing system built for Lycecom using Stripe. This project is forked from the [Bedrock Platform](https://github.com/bedrockio/bedrock-core), a full-stack application platform.

The application consists of two main services:
- **API** (`services/api`): Koa-based REST API with MongoDB using Mongoose
- **Web** (`services/web`): React/Vite frontend with Mantine UI components

The platform is multi-tenant ready and includes authentication, authorization, fixtures, background jobs, and comprehensive testing. As development progresses, Stripe integration will be added for payment processing functionality.

## Common Development Commands

### Running Services

```bash
# Run all services with Docker Compose (API, Web, MongoDB)
docker compose up

# Run API only (requires MongoDB running)
cd services/api
yarn start                    # Development with auto-reload
yarn start:production         # Production mode
yarn debug                    # Development with MongoDB query logging

# Run Web only
cd services/web
yarn start                    # Development server (http://localhost:2200)
yarn build                    # Production build
yarn static                   # Serve static production build
```

### Testing

```bash
# API tests (Jest)
cd services/api
yarn test                     # Run all tests
yarn test:watch               # Watch mode

# Web tests (Vitest)
cd services/web
yarn test                     # Run all tests
```

Tests MUST:
- Focus on critical flows and complex logic
- Validate observable behavior rather than implementation details
- Be colocated with source files using `.test.js` suffix

### Linting

```bash
# Both services use ESLint and Prettier
cd services/api
yarn lint

cd services/web
yarn lint
```

### Fixtures

```bash
cd services/api
yarn fixtures:reload          # Drop DB and reload all fixtures
yarn fixtures:load            # Load fixtures if DB is empty
yarn fixtures:export --models=User,Shop  # Export documents as fixtures
```

Fixtures are automatically loaded when `yarn start` finds an empty database.

### Documentation

```bash
cd services/api
yarn docs:generate           # Generate OpenAPI documentation from routes
```

## Development vs Production Environments

### Local Development

Local development uses `docker-compose.yml` which references `Dockerfile.dev` for both services:

```bash
# Start all services locally
docker compose up

# Rebuild after dependency changes
docker compose up --build

# Stop services
docker compose down
```

**Key characteristics:**
- Uses `Dockerfile.dev` (development-optimized with hot reloading)
- Mounts local source directories as volumes for live code updates
- Runs on localhost ports (2200, 2300, 27017)
- Auto-loads fixtures on empty database
- Default admin: `admin@bedrock.foundation` / `development.now`

**Alternative: Run without Docker**
```bash
# Requires Node.js 24.12.0 (use Volta for version management)
# Terminal 1 - MongoDB
docker run -p 27017:27017 mongo:8.0.9

# Terminal 2 - API
cd services/api
yarn start

# Terminal 3 - Web
cd services/web
yarn start
```

### Staging/Production Deployment

Production deployments use the [Bedrock CLI](https://github.com/bedrockio/bedrock-cli) and Kubernetes:

```bash
# Install Bedrock CLI (requires gcloud CLI and Terraform)
curl -s https://install.bedrock.io | bash

# Build production Docker images
bedrock cloud build           # Select services interactively
bedrock cloud build api       # Build specific service
bedrock cloud build api jobs  # Build sub-service (Dockerfile.jobs)

# Deploy to cloud
bedrock cloud deploy          # Deploy all services
bedrock cloud deploy api      # Deploy specific service
```

**Key characteristics:**
- Uses production `Dockerfile` (optimized, multi-stage builds)
- Deployed to Google Kubernetes Engine (GKE)
- Configuration in `deployment/environments/{staging,production}/`
- Managed secrets via Bedrock CLI (never commit secrets)
- See `deployment/README.md` for full provisioning and deployment docs

**Important:** Local development changes to `docker-compose.yml` and `Dockerfile.dev` do NOT affect production deployments.

## Architecture

### Service Ports

- **Web**: http://localhost:2200
- **API**: http://localhost:2300
- **MongoDB**: localhost:27017
- **API Docs**: http://localhost:2200/docs

### API Service (`services/api`)

**Key Directories:**
- `src/models/` - Mongoose models using [@bedrockio/model](https://github.com/bedrockio/model)
  - `definitions/*.json` - Model schemas in JSON format
  - `index.js` - Model exports and initialization
- `src/routes/` - API route handlers organized by resource
  - `__openapi__/` - OpenAPI documentation definitions
  - `auth/` - Authentication flows (password, OTP, passkey, federated)
- `src/utils/` - Middleware, helpers, and utilities
- `fixtures/` - Database seed data
- `scripts/` - CLI scripts for database, jobs, etc.
- `emails/` - Email templates (Mustache/Markdown)

**Core Dependencies:**
- `@bedrockio/model` - Mongoose extensions (validation, search, soft delete, access control)
- `@bedrockio/config` - Environment configuration from `.env`
- `@bedrockio/yada` - Request validation schemas
- `@bedrockio/fixtures` - Database seeding
- `@bedrockio/logger` - Structured logging with GCP integration
- `@bedrockio/templates` - Email template rendering

**Authentication Methods:**
- Password (with optional MFA: SMS, Email, Authenticator App)
- OTP (Email or SMS)
- Passkey (WebAuthn)
- Federated (Google, Apple)

Configuration in `services/api/.env` and login entrypoint in `services/web/src/App.js`.

### Web Service (`services/web`)

**Key Directories:**
- `src/screens/` - Page-level components
- `src/components/` - Reusable UI components
- `src/stores/` - React context stores (session management)
- `src/utils/` - Helpers for API calls, formatting, forms, etc.
- `src/hooks/` - Custom React hooks
- `src/docs/` - Markdown-powered API documentation portal

**Core Dependencies:**
- `@mantine/core` - UI component library (v8)
- `@bedrockio/router` - Client-side routing
- `react` (v19) and `react-dom`
- `vite` - Build tool and dev server

**Key Patterns:**
- Use `SearchProvider` component for paginated lists/tables
- Access session with `useSession()` hook or `withSession()` HOC
- Make API calls with `request()` from `utils/api`
- Environment variables exposed via `utils/env`

### Multi-Tenancy

The API supports multiple tenancy patterns:
- Each request has a "Default" organization
- Can be overridden with "Organization" header
- `authenticate()` middleware populates `ctx.state.organization`
- Use `requirePermissions('resource.write', 'organization')` to scope access

Example creating organization-scoped data:
```js
await Shop.create({
  organization: ctx.state.organization,
  ...ctx.request.body,
});
```

### Background Jobs

Jobs run via Yacron in a separate Docker container. Configuration in `jobs/*.yml`:

```yaml
defaults:
  timezone: America/New_York
jobs:
  - name: example
    command: node scripts/jobs/example.js
    schedule: '*/10 * * * *'
    concurrencyPolicy: Forbid
```

Build: `docker build -t bedrock-api-jobs -f Dockerfile.jobs .`

## Data Modeling Best Practices

When creating or modifying models in `services/api/src/models/definitions/`:

**Naming Conventions:**
- Dates end with `At` (e.g., `startsAt`, `createdAt`)
- Booleans prefix with `is`, `has`, `can`, `should` (e.g., `isActive`)
- Use positive booleans (`isActive` not `isInactive`)
- Reference fields named by model (`user` not `userId`) - may be populated
- Numeric values include units when not obvious (`amountCents`)
- Avoid abbreviations and generic names

**Array References:**
Avoid direct ObjectId arrays. Nest references to allow future metadata:

❌ Avoid:
```json
{
  "conditions": [{ "type": "ObjectId", "ref": "Condition" }]
}
```

✅ Prefer:
```json
{
  "conditions": [
    {
      "condition": { "type": "ObjectId", "ref": "Condition" }
    }
  ]
}
```

**Index Management:**
Store all indexes in `/indexes` folder, not in model definitions. Use `scripts/indexes/sync` to apply.

## Web Coding Conventions

From `services/web/.github/copilot-instructions.md`:

- **Formatting**: ESLint and Prettier
- **UI Components**: Use Mantine v8 whenever possible before creating custom components
- **Functions**: Use function declarations instead of arrow functions
- **Documentation**: Include JSDoc comments when appropriate
- **Lodash**: Use named imports for tree-shaking: `import { omit } from 'lodash-es';`

## Email Templates

Templates in `services/api/emails/`:
- Use Markdown or HTML with Mustache templating
- Layout in `layout.html` provides default styling
- Create buttons in Markdown: `**[Reset Password]({{{APP_URL}}}/reset-password?token={{token}})**`
- Unescape URLs: `{{{APP_URL}}}` not `{{APP_URL}}`

## Configuration

Both services use environment variables from `.env` files:
- **API**: `services/api/.env`
- **Web**: `services/web/.env`

See respective README files for full configuration options.

## Deployment

Deployment automation and provisioning documentation in `deployment/` directory.

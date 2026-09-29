# The Best Birthday Reminder

![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white)
![NestJS](https://img.shields.io/badge/NestJS-11-E0234E?logo=nestjs&logoColor=white)
![Expo](https://img.shields.io/badge/Expo-54-000020?logo=expo&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-4169E1?logo=postgresql&logoColor=white)
![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)
![Status: work in progress](https://img.shields.io/badge/status-work_in_progress-orange)

A cross-platform birthday reminder built as a TypeScript monorepo: an Expo (React Native) mobile app, a NestJS API, Supabase authentication and PostgreSQL through Prisma.

> **Status: early work in progress.** The foundation is in place (monorepo tooling, auth plumbing, database setup). The birthday features themselves are not built yet. [Project status](#project-status) spells out exactly what works today and what does not.

## Why

The goal is a focused mobile app for remembering the birthdays of people you care about. The repository is organised around a few deliberate choices:

- **One language end to end.** TypeScript in the mobile app, the API and the shared tooling.
- **Auth delegated, not reinvented.** Users sign in through Supabase (Google Sign-In); the API only verifies the JWT that Supabase issues.
- **A typed data layer.** Prisma over PostgreSQL, with a local database one `docker compose` away.
- **One repo, one toolchain.** pnpm workspaces and Turborepo drive install, build, lint and type-checking for every package.

## What exists today

- **Monorepo** with two apps (`api`, `mobile`) and shared ESLint and TypeScript config packages, managed with pnpm workspaces and Turborepo.
- **Mobile app** (Expo SDK 54, Expo Router): file-based tab navigation plus a modal route, NativeWind (Tailwind CSS) configured for styling, automatic light/dark theme, React Native New Architecture and React Compiler enabled.
- **Supabase client** (`apps/mobile/lib/supabase.ts`) with session persistence in AsyncStorage and token auto-refresh tied to the app's foreground/background state.
- **Google Sign-In component** (`apps/mobile/components/GoogleSignIn.tsx`) that obtains a Google ID token natively and exchanges it for a Supabase session with `signInWithIdToken`.
- **NestJS 11 API** with a global config module, a Passport JWT strategy that validates Supabase-issued bearer tokens (`SUPABASE_JWT_SECRET`, expired tokens rejected) and a global Prisma module.
- **Prisma 7** configured for PostgreSQL through `prisma.config.ts` and `DATABASE_URL`.
- **Docker Compose** service for a local PostgreSQL 15 instance.

## Tech stack

| Layer          | Technology                                                                           |
| -------------- | ------------------------------------------------------------------------------------ |
| Monorepo       | pnpm 9 workspaces, Turborepo 2                                                       |
| Language       | TypeScript                                                                           |
| Mobile         | Expo SDK 54, React Native 0.81, React 19.1, Expo Router 6, NativeWind 4              |
| Authentication | Supabase Auth (`@supabase/supabase-js`), `@react-native-google-signin/google-signin` |
| API            | NestJS 11, Passport (`passport-jwt`), `@nestjs/config`                               |
| Data           | PostgreSQL 15, Prisma 7                                                              |
| Testing        | Jest, Supertest                                                                      |
| Code quality   | ESLint 9, Prettier 3                                                                 |
| Local infra    | Docker Compose                                                                       |

## Architecture

```
.
├── apps/
│   ├── api/                    NestJS API
│   │   ├── prisma/schema.prisma    PostgreSQL datasource (no models yet)
│   │   └── src/
│   │       ├── auth/               JwtStrategy: verifies Supabase bearer tokens
│   │       └── prisma/             Global PrismaModule / PrismaService
│   └── mobile/                 Expo + Expo Router app
│       ├── app/                    File-based routes ((tabs)/ and a modal)
│       ├── components/             GoogleSignIn.tsx and Expo starter components
│       └── lib/supabase.ts         Supabase client
├── packages/
│   ├── ui/                     Turborepo starter React components (not used by the apps yet)
│   ├── eslint-config/          Shared ESLint presets (from the starter)
│   └── typescript-config/      Shared tsconfig bases (from the starter)
├── docker-compose.yml          Local PostgreSQL 15
├── pnpm-workspace.yaml
└── turbo.json
```

Intended data flow. Solid arrows have code behind them; dashed arrows are not connected yet (the mobile app has no API client, and no route is protected or backed by a model).

```mermaid
flowchart LR
    M["Expo mobile app"] -->|"Google ID token"| S["Supabase Auth"]
    S -->|"session / JWT"| M
    M -.->|"Bearer JWT"| A["NestJS API"]
    A -.->|"Prisma"| P[("PostgreSQL")]
```

## Getting started

### Prerequisites

- **Node.js 22 LTS.** Prisma 7 supports Node 20.19+, 22.12+ and 24+; the root `package.json` declares `>=18`, which is too permissive for it.
- **pnpm 9** (`corepack enable` picks the version from the `packageManager` field).
- **Docker**, for the local PostgreSQL instance.
- For the mobile app on a device or emulator: Android Studio and/or Xcode.
- For sign-in: a Supabase project and a Google OAuth web client ID.

### 1. Install

```bash
pnpm install
```

### 2. Configure environment variables

| Variable                           | File               | Purpose                                                                             |
| ---------------------------------- | ------------------ | ----------------------------------------------------------------------------------- |
| `DATABASE_URL`                     | `apps/api/.env`    | PostgreSQL connection string, read by Prisma. Required for Prisma commands.         |
| `SUPABASE_JWT_SECRET`              | `apps/api/.env`    | JWT secret of your Supabase project. Required: the API refuses to start without it. |
| `PORT`                             | `apps/api/.env`    | Optional, defaults to `3000`.                                                       |
| `EXPO_PUBLIC_SUPABASE_URL`         | `apps/mobile/.env` | Supabase project URL.                                                               |
| `EXPO_PUBLIC_SUPABASE_ANON_KEY`    | `apps/mobile/.env` | Supabase anon (public) key.                                                         |
| `EXPO_PUBLIC_GOOGLE_WEB_CLIENT_ID` | `apps/mobile/.env` | Google OAuth web client ID used by Google Sign-In.                                  |

The mobile app ships an example file; the API does not, so create it by hand. The `DATABASE_URL` below matches the credentials in `docker-compose.yml`.

```bash
cp apps/mobile/.env.example apps/mobile/.env   # then fill in your values

cat > apps/api/.env <<'EOF'
DATABASE_URL="postgresql://user:password@localhost:5432/birthday_reminder"
SUPABASE_JWT_SECRET="replace-with-your-supabase-jwt-secret"
EOF
```

`.env` files are git-ignored.

### 3. Start PostgreSQL and generate the Prisma client

```bash
docker compose up -d postgres
pnpm --filter api exec prisma generate
```

### 4. Run the API

```bash
pnpm --filter api start:dev
```

Once it boots, the API listens on `http://localhost:3000` and serves a placeholder `GET /` that returns `Hello World!`.

> **Heads-up:** the API does not boot yet. `PrismaService` still needs the Prisma 7 driver adapter, which is the first item on the [roadmap](#roadmap).

### 5. Run the mobile app

```bash
pnpm --filter mobile start      # Expo dev server
pnpm --filter mobile android    # native dev build on an Android emulator/device
pnpm --filter mobile ios        # native dev build on an iOS simulator/device
```

Google Sign-In is a native module, so it needs a development build (`android` / `ios`) rather than Expo Go.

## Scripts

Run from the repository root.

| Command            | What it does                                                                                                                                                   |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm build`       | `turbo run build`. Currently builds the API (`nest build`); the mobile app has no build script.                                                                |
| `pnpm dev`         | `turbo run dev`. Runs `dev` in workspaces that define it, which today is only the mobile app (`expo start`). Start the API with `pnpm --filter api start:dev`. |
| `pnpm lint`        | `turbo run lint` across all workspaces.                                                                                                                        |
| `pnpm check-types` | `turbo run check-types`. Only `@repo/ui` defines this task today.                                                                                              |
| `pnpm format`      | Prettier over `**/*.{ts,tsx,md}`.                                                                                                                              |

Per-app scripts (use `pnpm --filter <app> <script>`):

| App      | Scripts                                                                                                          |
| -------- | ---------------------------------------------------------------------------------------------------------------- |
| `api`    | `start`, `start:dev`, `start:debug`, `start:prod`, `build`, `lint`, `test`, `test:watch`, `test:cov`, `test:e2e` |
| `mobile` | `start` / `dev`, `android`, `ios`, `web`, `lint`, `reset-project`                                                |

Note that `api`'s `lint` script runs ESLint with `--fix`, so it rewrites files in place.

## Project status

| Area                                       | State                                                                                                     |
| ------------------------------------------ | --------------------------------------------------------------------------------------------------------- |
| Monorepo tooling (pnpm, Turborepo)         | Working: `pnpm install`, `pnpm build` (API) and the API unit test run cleanly.                            |
| Mobile app UI                              | Expo starter screens (Home, Explore, modal). No birthday screens yet.                                     |
| Mobile auth                                | Supabase client and Google Sign-In component exist but are not mounted in any screen.                     |
| API auth                                   | JWT strategy implemented and registered; no route uses a guard yet.                                       |
| API persistence                            | Prisma configured; `schema.prisma` has no models and there are no migrations.                             |
| Birthday domain (people, dates, reminders) | Not started.                                                                                              |
| Tests                                      | One placeholder unit test (`Hello World!`). The e2e test currently fails until the Prisma adapter is wired.                     |
| Docker                                     | Local PostgreSQL service works. The API image and its compose service (commented out) are not up to date. |

Parts of the scaffold still come from the Turborepo, Nest and Expo starters (`packages/ui`, the per-app READMEs, the root package name), and are next in line to be replaced.

## Roadmap

Planned, none of it implemented yet:

1. Make the API boot: configure `PrismaService` with the Prisma 7 driver adapter, clear the lint errors, fix the `start:prod` entry point and the Dockerfile for the pnpm workspace, and add CI (lint, type-check, test).
2. Data model and migrations for people and their birthdays.
3. REST endpoints for managing birthdays, protected with a JWT guard.
4. Mobile: sign-in screen, then birthday list, add and edit screens replacing the starter UI.
5. Reminders: scheduling and push notifications.
6. Working container image for the API and a deployment path.

## License

Released under the **GNU General Public License, Version 3 (29 June 2007)**. See [LICENSE](LICENSE) for the full text.

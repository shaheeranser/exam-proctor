# exam-proctor

An LTI 1.3 proctored exam platform that launches directly from Moodle. Teachers build quizzes inside the tool, students take them under browser-side proctoring, and results are graded and passed back to the LMS.

## Architecture

A single Node/Express service serves both the React client and the API, and acts as the LTI 1.3 tool (via `ltijs`). Supporting services run alongside it in Docker: MongoDB (data + LTI registrations), MinIO (incident snapshots), and Caddy (reverse proxy). See [SETUP.md](SETUP.md) for the cold-start walkthrough.

## Features

**Teacher**

- Build quizzes: questions, options, correct answers, per-question scores
- Draft/publish workflow with optional schedule (start/end) and attempt duration
- Student access control — course enrollment or explicit allowlist
- Manual open/close override
- Live dashboard of in-progress attempts (updates over SSE)
- Pause and resume individual attempts
- Review flagged attempts (uphold, dismiss, or grant a retest) and set final scores

**Student**

- Launched from Moodle and routed by LTI role
- Timed exam with automatic submission on timeout / tab close
- Browser-side proctoring: face detection, multiple-person detection, presence loss, and visibility/tab-switch guards
- Strike system with a configurable soft-violation limit
- Incident snapshots uploaded to MinIO; identity snapshot captured at start
- Dedicated terminated screen

**Grading**

- Automatic scoring and grade passback to Moodle via LTI AGS (with retry)

## Stack

- **Server:** Node.js, Express 5, TypeScript, `ltijs` (LTI 1.3), Mongoose/MongoDB, MinIO SDK
- **Client:** React 19, Vite, React Router, TypeScript; MediaPipe Tasks Vision + TensorFlow.js + face-api running in a web worker for detection
- **Infrastructure:** Docker Compose, Caddy, MongoDB 5, MinIO

## Project structure

```
server/src/
  controllers/   quiz, attempt, session, dashboard, pause, review, approval, passback, health
  routes/        API routes (quiz, attempt, session, health)
  models/        Mongoose schemas (Quiz, Attempt, Incident)
  lib/           minio, proctoring config, SSE, grade passback
  index.ts       LTI setup + Express bootstrap
client/src/
  shared/        API client, shared components, theme
  student/       eligibility gate, exam page, detection worker + guards
  teacher/       dashboard, live dashboard, quiz builder, review
scripts/deploy-ec2.sh
Dockerfile · docker-compose.yml · Caddyfile · SETUP.md
```

## Getting started

The full walkthrough — including the local Moodle/LTI environment and external tool registration — is in [SETUP.md](SETUP.md). In short:

```bash
cp .env.example server/.env   # fill in the values
docker compose up --build
```

The app listens on port 3000; Moodle is reached through the Caddy proxy.

For local development without Docker:

```bash
npm install
npm --prefix server install && npm --prefix client install
npm run dev                  # runs client + server concurrently
```

## Configuration

Key variables (see `.env.example`):

- `MONGO_URI` — MongoDB connection
- `MINIO_*` — snapshot storage
- `LTI_ENCRYPTION_KEY` — required; encrypts stored platform data
- `PLATFORM_*` — Moodle LTI registration values (filled after registering the tool in Moodle)

## Notes

- Requires two external Docker networks: `examnet` and `moodle-docker_default`.
- On LTI launch, instructors are routed to `/teacher`; everyone else to `/student`.

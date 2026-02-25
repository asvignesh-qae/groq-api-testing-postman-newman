# Groq API Testing — Postman & Newman

An automated API testing suite for validating Groq AI endpoints using Postman collections, Newman CLI runner, and GitHub Actions CI/CD.

---

## Table of Contents

- [Overview](#overview)
- [Project Structure](#project-structure)
- [Test Coverage](#test-coverage)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Running Tests](#running-tests)
- [CI/CD Pipeline](#cicd-pipeline)
- [Reports](#reports)
- [Models Tested](#models-tested)

---

## Overview

This project tests the [Groq API](https://console.groq.com/) (OpenAI-compatible) across four key capability areas:

| Area | Description |
|------|-------------|
| **LLM Chat Completions** | Code generation via multiple large language models |
| **Multilingual Translation** | Translation to Tamil, Japanese, and German |
| **Audio Transcription** | Speech-to-text using Whisper models |
| **Image OCR & Translation** | Text extraction and translation from images |

Tests run on a scheduled CI/CD pipeline every 8 hours and on every push/PR to `main`, with HTML reports generated automatically.

---

## Project Structure

```
groq-api-testing-postman-newman/
├── .github/
│   └── workflows/
│       └── newman-tests.yml          # GitHub Actions CI/CD workflow
├── postman-collection/
│   ├── groq.postman_collection.json  # Main Postman collection (15 requests)
│   └── groq.postman_environment.json # Environment variables template
├── newman-reports/                   # Output directory for local reports
├── .env                              # Local API keys (never committed)
├── .gitignore
└── package.json
```

---

## Test Coverage

The Postman collection **Gen AI - HA - Postman Collection** contains 15 requests organized into 4 groups:

### HA1 — LLM Code Generation (4 requests)
Tests code generation from a web page DOM using different models at `temperature: 0.1`.

| Request | Model |
|---------|-------|
| HA1 - S17 | `openai/gpt-oss-120b` |
| HA1 - S17 | `qwen/qwen3-32b` |
| HA1 - S17 | `meta-llama/llama-4-scout-17b-16e-instruct` |
| HA1 - S17 | `moonshotai/kimi-k2-instruct-0905` |

**Endpoint:** `POST https://api.groq.com/openai/v1/chat/completions`
**Assertions:** HTTP 200, response body validated against `groqResponseSchema` (stored in Postman environment)

---

### HA2 — Multilingual Translation (6 requests)
Tests translation of QA test steps into three languages using `top_p: 0.1`.

| Request | Language | Model |
|---------|----------|-------|
| HA2 - S21 | Tamil | `openai/gpt-oss-120b` |
| HA2 - S21 | Tamil | `meta-llama/llama-4-scout-17b-16e-instruct` |
| HA2 - S21 | Japanese | `openai/gpt-oss-120b` |
| HA2 - S21 | Japanese | `meta-llama/llama-4-scout-17b-16e-instruct` |
| HA2 - S21 | German | `openai/gpt-oss-120b` |
| HA2 - S21 | German | `meta-llama/llama-4-scout-17b-16e-instruct` |

**Assertions:** HTTP 200, JSON schema validation, language-specific keyword checks (e.g. `திறக்கவும்`, `入力し`, `die Anwendung`)

---

### HA3 — Audio Transcription (2 requests)
Tests Whisper models on an `.m4a` audio file using `multipart/form-data`.

| Request | Model |
|---------|-------|
| HA3 - S16 | `whisper-large-v3` |
| HA3 - S16 | `whisper-large-v3-turbo` |

**Endpoint:** `POST https://api.groq.com/openai/v1/audio/transcriptions`
**Assertions:** HTTP 200, JSON schema validation, content contains `"QA Automation"`

---

### HA4 — Image OCR & Translation (2 requests)
Tests vision models for multilingual text extraction from images with retry logic.

| Request | Input Type |
|---------|-----------|
| HA4 - S09 | Image URL |
| HA4 - S10 | Base64 encoded image |

**Features:**
- Exponential backoff retry (max 3 retries: 1s → 2s → 4s delay)
- Automatic retry on 5xx errors using collection variables

---

## Prerequisites

- [Node.js](https://nodejs.org/) v20+
- [npm](https://www.npmjs.com/)
- A Groq API key — get one at [console.groq.com](https://console.groq.com/)

---

## Installation

```bash
git clone https://github.com/asvignesh-qae/groq-api-testing-postman-newman.git
cd groq-api-testing-postman-newman
npm install
```

---

## Configuration

### Local Setup

Create a `.env` file in the project root (already in `.gitignore`):

```bash
GROQ_API_KEY=your_groq_api_key_here
```

Create a local environment file for Postman (also in `.gitignore`):

```
postman-collection/groq.postman_environment_local.json
```

Copy `groq.postman_environment.json` and replace the placeholder values with your actual API keys.

### GitHub Actions Secrets

For CI/CD, add these secrets in your GitHub repository settings under **Settings → Secrets and variables → Actions**:

| Secret | Description |
|--------|-------------|
| `GROQ_API_KEY` | Your Groq API key |
| `HUGGINGFACE_API_KEY` | HuggingFace API key (optional) |
| `OPENAI_API_KEY` | OpenAI API key (optional) |

---

## Running Tests

### Run with default environment (CI environment file)

```bash
npm run newman-test
```

### Run with local environment file

```bash
npm run newman:test:local
```

### Run LangFlow integration tests locally

```bash
npm run newman:langflow:local
```

### Run directly with Newman

```bash
npx newman run "postman-collection/groq.postman_collection.json" \
  -e "postman-collection/groq.postman_environment.json" \
  --env-var "GROQ_API_KEY=your_api_key" \
  -r htmlextra \
  --reporter-htmlextra-export "./newman-reports/report.html" \
  --reporter-htmlextra-skipEnvironmentVars "GROQ_API_KEY" \
  --reporter-htmlextra-skipHeaders "Authorization"
```

---

## CI/CD Pipeline

The GitHub Actions workflow ([.github/workflows/newman-tests.yml](.github/workflows/newman-tests.yml)) runs automatically:

| Trigger | When |
|---------|------|
| `push` | On every push to `main` |
| `pull_request` | On every PR targeting `main` |
| `schedule` | Every 8 hours (`0 */8 * * *`) |

**Workflow steps:**
1. Checkout repository
2. Setup Node.js 20 with npm cache
3. Install dependencies via `npm ci`
4. Run Newman tests with HTML reporter
5. Upload `test-report.html` as a build artifact (retained 30 days)

---

## Reports

HTML reports are generated using [newman-reporter-htmlextra](https://github.com/DannyDainton/newman-reporter-htmlextra).

**Features:**
- Visual pass/fail summary per request
- Request/response details
- Progress bar display
- Sensitive data masking — API keys and `Authorization` headers are redacted from reports

**In CI/CD:** Reports are published to GitHub Pages after every run on `main` and accessible at:

```
https://<user>.github.io/groq-api-testing-postman-newman/api/
```

**Deployment flow:**

```
Newman run → ./gh-pages/api/index.html
                    ↓
        upload-pages-artifact (./gh-pages)
                    ↓
           deploy-pages → https://<user>.github.io/groq-api-testing-postman-newman/api/
```

**Local reports:** Saved to `./newman-reports/` (excluded from version control).

---

## Models Tested

| Model | Provider | Use Case |
|-------|----------|----------|
| `openai/gpt-oss-120b` | OpenAI (via Groq) | Code generation, Translation |
| `qwen/qwen3-32b` | Alibaba | Code generation |
| `meta-llama/llama-4-scout-17b-16e-instruct` | Meta | Code generation, Translation |
| `moonshotai/kimi-k2-instruct-0905` | Moonshot AI | Code generation |
| `whisper-large-v3` | OpenAI (via Groq) | Audio transcription |
| `whisper-large-v3-turbo` | OpenAI (via Groq) | Audio transcription |

---

## License

ISC — see [package.json](package.json) for details.

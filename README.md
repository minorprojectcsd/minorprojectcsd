# MindCareX — Mental Health Telemedicine Platform

> **One platform. Five intelligent services. Real-time mental health insights during every consultation.**

MindCareX is a full-stack telemedicine platform built for mental health consultations. Doctors and patients connect via secure video sessions, and behind the scenes, a microservices architecture continuously analyses voice, facial expressions, and chat history to surface actionable clinical insights — live, during the session.

---

## 📦 Repositories

| Module | Repository | Description |
|--------|-----------|-------------|
| 🏗️ Backend | [mindcarex-backend](https://github.com/minorprojectcsd/mindcarex-backend) | Spring Boot REST API + WebSocket server |
| 🎙️ Audio API | [mindcarex-audio-api](https://github.com/minorprojectcsd/mindcarex-audio-api) | Voice stress analysis (SVC1) |
| 📋 Report API | [mindcarex-report-api](https://github.com/minorprojectcsd/mindcarex-report-api) | AI clinical report generator (SVC2) |
| 📷 Camera API | [mindcarex-camera-api](https://github.com/minorprojectcsd/mindcarex-camera-api) | Facial expression analysis (SVC3) |
| 💬 Chat API | [mindcarex-chat-api](https://github.com/minorprojectcsd/mindcarex-chat-api) | WhatsApp chat analyser |

---

## 🔗 The Connecting Thread

Every service in MindCareX shares a single purpose: **turning a video consultation into a clinically rich, data-backed session**.

The platform is built around a **shared Neon PostgreSQL database** and a **WebSocket-first design**. Data flows from the patient's microphone and camera in real time, is processed by two specialised analysis services, synthesised into a clinical report by an LLM, and then surfaced to the doctor's screen — all within the same session window.

```
Patient (browser)
  │
  ├── Audio chunks (every 7s)  ──▶  SVC1: Voice Analysis
  │                                        │ stress score, transcript
  │                                        ▼
  ├── Video frames (every 5–10s) ▶  SVC3: Camera Analysis
  │                                        │ face stress, expressions
  │                                        ▼
  │                               Combined stress score (voice 55% + face 45%)
  │                                        │
  │                                        ▼
  │                               SVC2: Report Generator
  │                                        │ clinical report + guardian message
  │                                        ▼
  └── WhatsApp export (upload) ──▶ Chat Analyser
                                           │ sentiment, risk flags, patterns
                                           ▼
                                  Spring Boot Backend
                                  (JWT auth · appointments · email notifications)
```

All services write to and read from the **same shared Neon PostgreSQL instance**. WebSockets broadcast results to the doctor's dashboard in under 100ms.

---

## 🧩 Services at a Glance

| Service | Repo | Stack | Port | Role |
|---------|------|-------|------|------|
| **Backend** | [mindcarex-backend](https://github.com/minorprojectcsd/mindcarex-backend) | Spring Boot 3.5 / Java 21 | 8080 | Auth, appointments, sessions, email, WebRTC signalling |
| **Audio API** | [mindcarex-audio-api](https://github.com/minorprojectcsd/mindcarex-audio-api) | FastAPI / Python | 8000 | Acoustic stress scoring + Whisper transcription |
| **Report API** | [mindcarex-report-api](https://github.com/minorprojectcsd/mindcarex-report-api) | FastAPI / Python | 8001 | LLaMA 3.3 clinical report + guardian message |
| **Camera API** | [mindcarex-camera-api](https://github.com/minorprojectcsd/mindcarex-camera-api) | FastAPI / Python | 8002 | Facial expression stress scoring via ViT model |
| **Chat API** | [mindcarex-chat-api](https://github.com/minorprojectcsd/mindcarex-chat-api) | Flask / Python | 5000 | WhatsApp export sentiment, risk flags, word patterns |

---

## 🏗️ Spring Boot Backend · [mindcarex-backend](https://github.com/minorprojectcsd/mindcarex-backend)

The backbone of the platform. Handles everything outside of AI analysis.

**Key responsibilities:**
- JWT-secured REST API with role-based access (Doctor / Patient / Admin)
- Appointment booking, cancellation, and status tracking
- Video session lifecycle (start → active → end)
- STOMP/WebSocket server for real-time chat and WebRTC signalling
- Automated email notifications via Brevo or Gmail SMTP (Thymeleaf templates)
- Scheduled 10-minute pre-session reminders (cron every minute)
- Reads `guardian_message` from SVC2 and emails it to the patient's emergency contact

**Data stores:** PostgreSQL (users, appointments, sessions, email logs) + MongoDB (chat messages with 24-hour TTL auto-delete)

**Live endpoints:**
```
REST:   https://mindcarex-backend.onrender.com
Health: https://mindcarex-backend.onrender.com/health
```

---

## 🎙️ Audio API — Voice Analysis · [mindcarex-audio-api](https://github.com/minorprojectcsd/mindcarex-audio-api)

Processes the patient's audio every 7 seconds throughout the session and pushes results to the doctor's screen in real time.

**How it works:**
1. Frontend sends a 7-second audio chunk (WebM / WAV / MP3)
2. 13 acoustic features are extracted via `librosa` (pitch, entropy, silence ratio, MFCCs, zero-crossing rate, etc.)
3. Speech is transcribed by **Groq Whisper**
4. A stress score (0–100) is computed from the acoustic features
5. Result is saved to Neon and broadcast via WebSocket (`/api/voice/{id}/live`)

**Stress score formula (acoustic only):**
```
stress = pitch_variability × 0.20  +  silence_pattern × 0.15
       + spectral_entropy  × 0.15  +  pitch_mean       × 0.15
       + zero_crossing     × 0.15  +  rms_energy       × 0.10
       + spectral_centroid × 0.10
```

| Score | State | Colour |
|-------|-------|--------|
| 72–100 | High Stress | 🔴 Red |
| 50–71 | Moderate Stress | 🟠 Orange |
| 30–49 | Mild Stress | 🟡 Yellow |
| 0–29 | Calm / Relaxed | 🟢 Green |

**Key env vars:** `DATABASE_URL`, `GROQ_API_KEY`

---

## 📋 Report API — Clinical Report Generator · [mindcarex-report-api](https://github.com/minorprojectcsd/mindcarex-report-api)

Runs once, after the session ends. Reads SVC1's output and generates a full clinical report using an LLM.

**How it works:**
1. Spring Boot calls `POST /api/report/generate` with the completed session ID
2. SVC2 reads the transcript and summary from Neon (written by SVC1)
3. Two prompts are sent to **Groq LLaMA 3.3 70B**:
   - A structured clinical report (JSON)
   - A plain-English guardian message (3–4 paragraphs)
4. Both are saved to Neon
5. Spring Boot retrieves the `guardian_message` and emails it to the patient's emergency contact

**Report sections:** session overview · stress analysis · vocal indicators · emotional state · risk assessment · recommendations · follow-up plan

**Behaviour notes:**
- Idempotent — calling generate twice returns the cached report on the second call
- Graceful fallback if no audio was captured (microphone failure scenario)

**Key env vars:** `DATABASE_URL`, `GROQ_API_KEY`

---

## 📷 Camera API — Facial Expression Analysis · [mindcarex-camera-api](https://github.com/minorprojectcsd/mindcarex-camera-api)

Analyses the patient's facial expressions every 5–10 seconds, in parallel with voice analysis.

**How it works:**
1. Frontend captures a JPEG frame from the patient's webcam
2. Frame is sent to **HuggingFace `trpakov/vit-face-expression`** (ViT-B/16, FER2013 + AffectNet trained)
3. 7 expression scores are returned (angry, disgusted, fearful, happy, neutral, sad, surprised)
4. Face stress score (0–100) is computed from weighted expression scores
5. If a voice session ID is linked, a **combined stress score** is computed
6. Result is saved to Neon and broadcast via WebSocket (`/api/camera/{id}/live`)

**Face stress formula:**
```
face_stress = fearful×1.00 + angry×0.85 + disgusted×0.70
            + sad×0.55 + surprised×0.30 + neutral×0.05 + happy×(−0.20)
```

**Combined score (voice + face):**
```
combined = 0.55 × voice_stress + 0.45 × face_stress
```
Risk auto-escalates to HIGH if both modalities independently flag high risk.

> ⚠️ **March 2026 note:** HuggingFace deprecated `api-inference.huggingface.co`. All calls must use `router.huggingface.co/hf-inference/models/`. The HF token must be Fine-grained with "Make calls to serverless Inference Providers" enabled.

**Key env vars:** `DATABASE_URL`, `HF_API_TOKEN`

---

## 💬 Chat API — WhatsApp Analyser · [mindcarex-chat-api](https://github.com/minorprojectcsd/mindcarex-chat-api)

A standalone Flask module doctors can use to analyse a patient's WhatsApp chat export before or after a consultation.

**How it works:**
1. Doctor uploads a `.txt` WhatsApp export
2. Messages are parsed (Android and iOS formats both supported) into a Pandas DataFrame
3. Two sentiment engines run on every message: **VADER** (primary, social-text optimised) and **TextBlob** (secondary, adds subjectivity score)
4. Keyword-based risk scanning flags concerning language at three severity levels
5. Activity patterns, word frequency, and emoji analysis are computed
6. Results are cached in Neon for 120 minutes

**Risk levels:**
| Level | Keywords (sample) |
|-------|-------------------|
| 🔴 High | kill, suicide, threat, danger, abuse |
| 🟠 Medium | fight, frustrated, argument, scam, lie |
| 🟡 Low | sad, alone, worried, stressed, cry |

> ⚠️ Risk detection is keyword-based — it flags presence, not context. Always treat as a signal to review, not a clinical diagnosis.

**Key env vars:** `DATABASE_URL`, `SECRET_KEY`

---

## 🗄️ Shared Infrastructure

All services are connected through a common layer:

| Resource | Used by | Purpose |
|----------|---------|---------|
| **Neon PostgreSQL** | All services | Primary data store; SVC1 writes, SVC2/SVC3 read and write |
| **MongoDB Atlas** | Backend | Session chat messages (24h TTL auto-delete) |
| **WebSocket (STOMP)** | Backend, SVC1, SVC3 | Real-time push to doctor's dashboard |
| **Groq API** | SVC1, SVC2 | Whisper STT + LLaMA 3.3 70B report generation |
| **HuggingFace Inference** | SVC3 | ViT face expression model |
| **Brevo / Gmail SMTP** | Backend | Appointment and guardian email notifications |
| **Render (free tier)** | All services | Deployment platform |

---

## 🚀 Quick Start (Local)

```bash
# 1. Spring Boot backend
git clone https://github.com/minorprojectcsd/mindcarex-backend.git
cd mindcarex-backend
cp .env.example .env   # fill in DB, JWT, email vars
mvn spring-boot:run    # → http://localhost:8080

# 2. Voice analysis
git clone https://github.com/minorprojectcsd/mindcarex-audio-api.git
cd mindcarex-audio-api
pip install -r requirements.txt
uvicorn main:app --reload --port 8000

# 3. Report generator
git clone https://github.com/minorprojectcsd/mindcarex-report-api.git
cd mindcarex-report-api
pip install -r requirements.txt
uvicorn main:app --reload --port 8001

# 4. Camera analysis
git clone https://github.com/minorprojectcsd/mindcarex-camera-api.git
cd mindcarex-camera-api
pip install -r requirements.txt
uvicorn main:app --reload --port 8002

# 5. Chat analyser
git clone https://github.com/minorprojectcsd/mindcarex-chat-api.git
cd mindcarex-chat-api
pip install -r requirements.txt
python -m textblob.download_corpora
python app.py           # → http://localhost:5000
```

Health checks:
```bash
curl http://localhost:8080/health   # backend
curl http://localhost:8000/health   # svc1
curl http://localhost:8001/health   # svc2
curl http://localhost:8002/health   # svc3
curl http://localhost:5000/health   # chat analyser
```

---

## 🔒 Security

- JWT authentication (stateless, 24-hour expiry)
- Role-based access: `ROLE_ADMIN` / `ROLE_DOCTOR` / `ROLE_PATIENT`
- BCrypt password hashing (10 rounds)
- CORS configured per environment via `ALLOWED_ORIGINS`
- HTTPS / WSS in production; STARTTLS on port 587 for email
- MongoDB chat messages auto-deleted after 24 hours via TTL index

---

## 🌐 Production URLs

| Service | URL |
|---------|-----|
| Frontend | https://mindcarex.vercel.app |
| Backend API | https://mindcarex-backend.onrender.com |
| Health check | https://mindcarex-backend.onrender.com/health |

> **Render free tier note:** Services sleep after inactivity. Use a cron job (e.g. cron-job.org) pinging `/health` every 5 minutes to keep them warm.

---

## 📬 Contact & Acknowledgements

**Issues / feedback:** minorprojectcsd@gmail.com

| Repo | Link |
|------|------|
| Backend | [mindcarex-backend](https://github.com/minorprojectcsd/mindcarex-backend) |
| Audio API | [mindcarex-audio-api](https://github.com/minorprojectcsd/mindcarex-audio-api/) |
| Report API | [mindcarex-report-api](https://github.com/minorprojectcsd/mindcarex-report-api/) |
| Camera API | [mindcarex-camera-api](https://github.com/minorprojectcsd/mindcarex-camera-api/) |
| Chat API | [mindcarex-chat-api](https://github.com/minorprojectcsd/mindcarex-chat-api/) |

Built with: Spring Boot · FastAPI · Flask · PostgreSQL (Neon) · MongoDB Atlas · Groq · HuggingFace · Brevo · Render

*Built by the MindCareX Team — minorprojectcsd *

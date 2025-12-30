# HackChrono System Design

This document provides a comprehensive system design for the HackChrono platform, covering architecture, key components, service interactions, data flow, and operational considerations. It consolidates details from the codebase and existing guides to serve as a single reference for developers and maintainers.

## Overview

HackChrono is an agricultural commerce and advisory platform with:
- A modern React frontend (Vite) for buyers and sellers
- A Node.js/Express backend with MongoDB for core marketplace features
- Python FastAPI microservices providing AI features:
  - Crop Recommendation (port 8000)
  - Yield Prediction (port 8001)
  - Plant Disease Detection (port 8002)
- Real-time collaboration via Socket.IO
- Voice assistant for navigation and intent detection
- Stripe integration for payments

### Port Map
- Frontend: 5173 (Vite dev server)
- Backend: 5000 (Express + Socket.IO)
- Crop Recommendation: 8000 (FastAPI)
- Yield Prediction: 8001 (FastAPI)
- Disease Detection: 8002 (FastAPI)

## Components & Responsibilities

### Frontend (React, Vite)
- Location: `frontend/`
- Notable modules:
  - Seller AI page: `frontend/src/pages/seller/AI.jsx` – UI for all AI features
  - API client: `frontend/src/lib/api.js` – base URL selection, JWT headers
  - Voice Assistant: integrated in seller layout, uses Web Speech API for speech and speech synthesis
- Dependencies: React 19, React Router, Socket.IO client, i18next, Stripe.js

### Backend API (Node.js/Express)
- Entry: `backend/src/server.js`
- Responsibilities:
  - REST endpoints for auth, listings, orders, cart, buyer/seller flows, Stripe payments
  - AI proxy endpoints under `/api/ai/` that call Python services
  - Voice assistant endpoints under `/api/voice/`
  - Socket.IO server for real-time features (e.g., negotiation updates)
  - MongoDB persistence via Mongoose
  - CORS with multi-origin support via `CLIENT_ORIGIN` env
- Key routes:
  - AI: `backend/src/routes/ai.js`
  - Voice: `backend/src/routes/voice.js`
  - Others: `backend/src/routes/*`
- Auth middleware: `backend/src/middleware/auth.js` – JWT verification via `Authorization: Bearer <token>`
- Persistence model for AI results: `backend/src/models/AIResult.js`

### AI Microservices (Python FastAPI)
- Location: `crop-prediction/`
- Services:
  - Crop Recommendation (port 8000): `main.py` (started via `start_server.py`)
  - Yield Prediction (port 8001): `yield_api.py` (started via `start_yield_server.py`)
  - Disease Detection (port 8002): `disease_api.py` (started via `start_disease_server.py`)
- External integrations:
  - OpenWeatherMap API (Crop Recommendation): `.api_key.txt` required
- Models and datasets:
  - Crop Recommendation dataset and preprocessing under `crop-prediction/data/` and `Crop-Yield-Prediction-using-Machine-Learning-Algorithms/`
  - Disease Detection model file: `crop-prediction/plant_disease_model_1.pt` (download required)

### Database (MongoDB)
- Used by backend for users, listings, orders, carts, and AI results
- Connection configured in `backend/src/server.js` via `MONGODB_URI`
- Common collection example: `AIResult` with fields `userId`, `type`, `input`, `result`, and timestamps

### Real-Time (Socket.IO)
- Enabled in `backend/src/server.js`
- Event channels include negotiation updates (`neg:update`, `neg:upsert`) broadcast to the `negotiations` room

### Voice Assistant
- Backend intent routes: `/api/voice/intent`, `/api/voice/commands`
- Pattern-based recognition (English + Hindi), optional GPT-based recognition if `OPENAI_API_KEY` is present
- Actions include navigation, help, and search; responses include friendly system messages
- Frontend UI includes a floating microphone button and voice feedback via Web Speech Synthesis

### Payments (Stripe)
- Server-side integration via `backend/src/routes/stripe.js`
- Frontend uses `@stripe/react-stripe-js` and `@stripe/stripe-js`

## API Surface (Key Endpoints)

### Health
- `GET /api/health` → `{ ok: true }`

### AI
- `POST /api/ai/crop-recommendation`
  - Body: `{ nitrogen, phosphorous, potassium, ph, state, district, month }`
  - Auth: JWT required
  - Calls `http://localhost:8000/predict/`

- `POST /api/ai/yield-prediction`
  - Body: `{ area, rainfall, fertilizer, crop, state }`
  - Auth: JWT required
  - Calls `http://localhost:8001/predict-yield/` with ML; falls back to heuristic if service down

- `POST /api/ai/disease-detection`
  - Body: `{ image: <base64 data URL> }`
  - Auth: JWT required
  - Calls `http://localhost:8002/detect-base64/`

- `GET /api/ai/results?limit=20&type=YIELD_PREDICTION|CROP_RECOMMENDATION|DISEASE_DETECTION`
  - Returns recent persisted results for the authenticated user

### Voice
- `POST /api/voice/intent`
  - Body: `{ text, currentRoute, language }`
  - Returns `{ intent, action, route, message }`
- `GET /api/voice/commands` → static list of supported phrases

### Core Marketplace
- `auth`, `listings`, `orders`, `seller`, `buyer`, `cart`, `payments` routes under `/api/*` (see `backend/src/routes/`)

## Request Flows

### Crop Recommendation
1. Frontend submits soil and location parameters from `AI.jsx`
2. Backend `/api/ai/crop-recommendation` validates and forwards to FastAPI `:8000/predict/`
3. FastAPI computes recommendation using soil NPK, pH, rainfall, and climate (OpenWeatherMap) → returns crop name
4. Backend saves `AIResult` and returns a friendly message and crop

### Yield Prediction
1. Frontend submits area, rainfall, fertilizer, crop, state from `AI.jsx`
2. Backend `/api/ai/yield-prediction` forwards to FastAPI `:8001/predict-yield/`
3. FastAPI estimates yield using historical stats and input parameters → returns `{ yieldCategory, estimatedYield, yieldPerAcre, confidence, recommendations }`
4. If FastAPI is unavailable, backend performs a heuristic fallback and returns a simplified prediction
5. Backend saves `AIResult`

### Disease Detection
1. Frontend uploads leaf image; browser converts to base64 data URL
2. Backend `/api/ai/disease-detection` forwards to FastAPI `:8002/detect-base64/`
3. FastAPI loads CNN model (`plant_disease_model_1.pt`), predicts disease and confidence, and provides treatment suggestions
4. Backend saves `AIResult`

### Voice Intent
1. Frontend captures speech → text via Web Speech API
2. Backend `/api/voice/intent` recognizes intent via pattern matching or GPT
3. Frontend executes action: navigate, show info, or search

### Payments
1. Frontend initializes Stripe Elements
2. Backend creates payment intents and processes checkout

## Configuration & Environment

### Backend (`backend`)
- Env vars:
  - `PORT` (default `5000`)
  - `MONGODB_URI` (default `mongodb://localhost:27017/digikhet`)
  - `CLIENT_ORIGIN` (comma-separated origins; `*` allowed)
  - `JWT_SECRET` (default `dev-secret`)
  - `PYTHON_API_URL` (default `http://localhost:8000`)
  - `YIELD_API_URL` (default `http://localhost:8001`)
  - `DISEASE_API_URL` (default `http://localhost:8002`)
  - `OPENAI_API_KEY` (optional, enables GPT-based voice intent recognition)

### Python (`crop-prediction`)
- `.api_key.txt` with OpenWeatherMap API key (used by Crop Recommendation)
- Model file for disease detection: `plant_disease_model_1.pt` (download as documented)
- Requirements: see `crop-prediction/requirements.txt` and `requirements-minimal.txt`

### Frontend (`frontend`)
- `VITE_API_URL` optional override (auto-maps localhost frontend to backend `http://localhost:5000`)

### CORS
- Backend uses a dynamic origin list from `CLIENT_ORIGIN`; supports wildcard `*`
- Socket.IO server mirrors CORS policy

## Dependencies (Key)

### Backend
- `express`, `mongoose`, `cors`, `dotenv`, `jsonwebtoken`, `socket.io`, `stripe`

### Frontend
- `react`, `react-router-dom`, `socket.io-client`, `i18next`, `@stripe/react-stripe-js`, `@stripe/stripe-js`

### Python
- `fastapi`, `uvicorn`, `pydantic`, `pandas`, `numpy`, `torch`, `torchvision`, `Pillow`

## Operations & Deployment

### Local Development
See `START_ALL_SERVICES.md` and `QUICK_START.md` for detailed steps. Typical commands:

```bash
# Terminal 1 - Crop Recommendation (8000)
cd crop-prediction
python start_server.py

# Terminal 2 - Yield Prediction (8001)
cd crop-prediction
python start_yield_server.py

# Terminal 3 - Disease Detection (8002)
cd crop-prediction
python start_disease_server.py

# Terminal 4 - Backend (5000)
cd backend
npm run dev

# Terminal 5 - Frontend (5173)
cd frontend
npm run dev
```

### Health Checks
- Backend: `GET /api/health` → `{ ok: true }`
- Crop Recommendation: `GET http://localhost:8000` → basic info
- Yield Prediction: `GET http://localhost:8001` → service info
- Disease Detection: `GET http://localhost:8002` → service info

### Logs
- Python services output startup banners and request logs
- Backend logs server start and errors, and broadcasts Socket.IO events
- Frontend shows network requests in dev tools

## Security
- JWT authentication enforced via `requireAuth` for protected routes
- Input validation on backend AI routes; error handling and fallbacks
- CORS origin filtering with credentials support
- Recommended: rate limiting on voice routes, stronger validation and sanitization

## Scaling & Resilience
- Horizontally scale backend and Python services behind a reverse proxy/load balancer
- Consider separate GPU-enabled node for disease detection (PyTorch)
- Use caching (e.g., Redis) for frequent ML outputs and weather data
- Circuit breakers and fallbacks (already in place for yield prediction)
- Queue long-running ML jobs if needed

## Observability
- Persist AI outputs via `AIResult` for audit and analytics
- Add metrics and tracing (e.g., Prometheus, OpenTelemetry) in future iterations

## Testing
- Backend smoke test: `curl http://localhost:5000/api/health`
- Crop Recommendation:
  ```bash
  curl -X POST "http://localhost:8000/predict/" \
    -H "Content-Type: application/json" \
    -d '{
      "nitrogen": 90,
      "phosphorous": 42,
      "potassium": 43,
      "ph": 6.5,
      "state": "Punjab",
      "district": "Ludhiana",
      "month": "Jun-Sep"
    }'
  ```
- Yield Prediction:
  ```bash
  curl -X POST "http://localhost:8001/predict-yield/" \
    -H "Content-Type: application/json" \
    -d '{
      "area": 5,
      "rainfall": 800,
      "fertilizer": 150,
      "crop": "Rice",
      "state": "Punjab"
    }'
  ```
- Disease Detection (base64): see frontend `AI.jsx` to generate image preview; or POST to `:8002/detect/` with multipart

## Known Constraints & Future Enhancements
- Disease model file must be manually downloaded (`plant_disease_model_1.pt`)
- Crop Recommendation depends on OpenWeatherMap API key and dataset coverage
- Add robust error reporting and retries between backend and Python services
- Deploy advanced LSTM/RNN yield model in production
- Extend Voice Assistant with Deepgram or custom STT for improved accuracy
- Add multi-language support site-wide (i18n already present)

## File References
- Backend server: `backend/src/server.js`
- AI routes: `backend/src/routes/ai.js`
- Voice routes: `backend/src/routes/voice.js`
- Auth middleware: `backend/src/middleware/auth.js`
- AI result model: `backend/src/models/AIResult.js`
- Frontend AI page: `frontend/src/pages/seller/AI.jsx`
- Frontend API client: `frontend/src/lib/api.js`
- Crop Recommendation: `crop-prediction/start_server.py` (`main.py`)
- Yield Prediction: `crop-prediction/yield_api.py`, `crop-prediction/start_yield_server.py`
- Disease Detection: `crop-prediction/disease_api.py`, `crop-prediction/start_disease_server.py`
- Getting started: `QUICK_START.md`, `START_ALL_SERVICES.md`, `INTEGRATION_GUIDE.md`

---
This system design should help onboard contributors and guide architectural decisions as the platform evolves.

# Agentic CCTV Surveillance

**Agentic CCTV Surveillance** is a state-of-the-art, agentic, real-time video surveillance analysis platform. Designed to replace lenient, legacy rule-based systems and passive human monitors, Agentic CCTV Surveillance relies on a "Guilty Until Proven Innocent" paradigm. 

It uses a dual-agent architecture—combining a **Narrative Builder** and a **Reasoner**—to forensically track subjects, analyze behaviors across time, classify risk levels, and dispatch real-time alerts.

---

## Key Features

### 1. Dual-Agent Architecture
Agentic CCTV Surveillance employs two concurrently running LLM agents per video segment:
* **Narrative Builder**: Acts as a forensic narrator. It maintains a running state of exact hand positions, pocket states, item interactions, and body language across 2-minute video chunks, outputting a precise 4-5 sentence atomic reconstruction.
* **Reasoner**: Acts as a paranoid security evaluator. It processes the visual context, the operator-defined Danger Zone logic, and the scene history to assign a continuous risk score between 0.0 and 1.0, generating a detailed explanation of potential threats.

### 2. Race Condition & Memory Context Handling
Real-time feeds can outpace ML inferences. Agentic CCTV Surveillance simulates a live CCTV buffer:
* When the ML pipeline falls behind the live video stream (i.e. > 3 chunks or 6 minutes pending), the pipeline elegantly degrades to a **tool-less mode** using extracted frames, sacrificing deep API analysis for sub-second synchronization.
* **Scene History Persistence**: Both agents read from an accumulated historical log. If a subject returns in chunk 5 after appearing in chunk 1, the agents optionally execute a `view_historical_clip` tool to visually cross-reference their past vs. current state (e.g., *did their bag get fuller?*).

### 3. Familiar Faces (InsightFace)
Upload reference images of employees, known offenders, or VIPs. Using the **ArcFace (`buffalo_l`) local ONNX model**, the system seamlessly extracts face embeddings and identifies individuals in frames with sub-second latency. Identifications are passed directly to the LLM's prompt context to aid reasoning natively.

### 4. Danger Zones & Directives
Store managers can dynamically upload images defining a "Danger Zone" (e.g., "The locked electronics cabinet") and provide a textual directive. Any activity matching this definition is immediately prioritized by the Reasoner, automatically triggering a **maximum risk score (>= 0.8)** alert.

### 5. Semantic Search (Gemini Embeddings)
Ever need to find exactly when a person wearing a specific jacket entered? All atomic reconstructions are embedded and stored in an asynchronous SQLite vector format. Natural language queries (e.g., "Person acting nervous looking at the cameras") map directly to the specific video chunk through cosine similarity.

### 6. Real-Time UI Dashboard (WebSocket Stream)
A cinematic dark-mode React application streams all AI outputs instantly. Track risk scores on a live timeline, scrub through video segments synced with LLM activities, and read the "Agent Brain" logs as they generate real-time thoughts.

---

## Technical Visualizations

### Architecture Overview

![System Architecture](Docs/architecture/System%20Architecture.png)

### System Design & Pipeline Flow

![System Flow](Docs/architecture/System%20Flow.png)

### Context Engineering & State Management

![Context Engineering](Docs/architecture/Context%20Engineering.png)

---

## System Architecture Details

* **Backend**: Python 3.10+, FastAPI, `aiosqlite` (SQLite), WebSockets, InsightFace (onnxruntime).
* **AI Provider**: Google GenAI API (`gemini-3-flash-preview` and `gemini-embedding-001`) OR Local Models (LM Studio/GGUF).
* **Video Pipeline**: `ffmpeg` for splitting standard video files into simulated 2-minute live ingestion chunks. 
* **Frontend**: React 19, Vite, TailwindCSS v4, `lucide-react`.

---

## Prerequisites

Before you begin, ensure you have the following installed on your host system:
1. **Python 3.10** or higher
2. **Node.js 18** or higher
3. **FFmpeg**: Required for backend video chunking and frame extraction.
   * *Ubuntu/Debian:* `sudo apt install ffmpeg`
   * *macOS:* `brew install ffmpeg`
   * *Windows:* Download via `winget install ffmpeg` or the official website.
4. **C++ Build Tools**: Required by `insightface`.

---

## Installation & Setup

### 1. Clone the Repository
```bash
git clone <your-repo-url>
cd Agentic-CCTV-Surveillance
```

### 2. Backend Setup
Navigate to the `Backend` directory and set up a virtual environment.
```bash
cd Backend
python3 -m venv .venv
source .venv/bin/activate  # On Windows use: .venv\Scripts\activate
pip install -r requirements.txt
```

### 3. Frontend Setup
Open a new terminal, navigate to the `Frontend` directory.
```bash
cd Frontend
npm install
```

---

## Configuration (.env)

In the `Backend/` directory, create or edit the `.env` file to configure your AI Provider.

### Option A: Using Google Gemini (Default, Recommended)
Provides the full agentic loop capability including file-upload vision and tool-calling (`view_historical_clip`).
```env
AI_PROVIDER=gemini
GEMINI_API_KEY=your_actual_api_key_here
```

### Option B: Using Local Models (LM Studio / llama.cpp)
Run entirely offline on your hardware using quantized GGUF vision models (e.g., Llama-3.2-Vision). The system will automatically use the "tool-less" mode using hardcoded keyframe extraction to prevent VRAM explosion.

1. Open **LM Studio**.
2. Download a Vision-capable model (like `Llava` or `Llama-3.2-Vision`).
3. Start the Local Server on port `1234`.
4. Edit the `.env` file:
```env
AI_PROVIDER=local
LOCAL_ENDPOINT_URL=http://localhost:1234
# Example model identifier below:
LOCAL_MODEL_ID=llama-3.2-11b-vision-instruct
LOCAL_MAX_FRAMES=8
```

---

## Running the Application

You must run both the backend and frontend simultaneously. The SQLite database (`dssa.db`) is automatically initialized upon the first backend start.

**Terminal 1: Start the Backend (FastAPI)**
```bash
cd Backend
source .venv/bin/activate
python app.py
```
*The backend will be available at `http://localhost:8000`*

**Terminal 2: Start the Frontend (React)**
```bash
cd Frontend
npm run dev
```
*The frontend will be available at `http://localhost:3000`*


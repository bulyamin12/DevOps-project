# Local AI Chat — Flask + CrewAI + Ollama

A self-hosted AI assistant chat web app. It uses **Flask** as the web server and **CrewAI** to call language models served by **Ollama** (ideally running in Docker), so no data has to leave your machine.

It handles questions in both Thai and English, and lets you switch between a general-purpose chat model and a coding model.

---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Architecture and Flow](#architecture-and-flow)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Usage](#usage)
- [File-by-File Details](#file-by-file-details)
- [Supported Models](#supported-models)
- [Customization](#customization)
- [Current Status and Recommended Fixes](#current-status-and-recommended-fixes)
- [Troubleshooting](#troubleshooting)

---

## Features

- Chat with an AI through a dark-themed web page that works on desktop and mobile (responsive)
- Two selectable models: general conversation and coding help
- Answers in Thai or English, matching the language of the question
- Keeps recent conversation history in the browser (up to 10 messages)
- "Clear chat" button to start a fresh conversation
- Shows an "AI is thinking..." status and disables the send button while waiting
- Protects against HTML injection in messages (uses `textContent`, not `innerHTML`)
- Includes a terminal (CLI) chat mode in `general.py`

## Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Python 3.12 |
| Web framework | Flask 3.1 |
| AI orchestration | CrewAI 1.15 (`Agent`, `Task`, `Crew`, `LLM`) |
| LLM runtime | Ollama (in Docker) |
| Models | `llama3.2:3b`, `qwen2.5-coder:1.5b` |
| Frontend | HTML, CSS, vanilla JavaScript |
| Database (example in `app.py`) | SQLite via SQLAlchemy |

## Project Structure

```
project dev/
├── app.py              # Flask app (currently a store demo — see "Current Status")
├── general.py          # AI logic via CrewAI + Ollama, plus the CLI mode
├── requirements.txt    # Dependencies
├── templates/
│   └── index.html      # Chat web page
├── static/
│   ├── script.js       # Browser logic (send messages, render, clear chat)
│   └── style.css       # Page styling (dark theme)
└── .venv/              # Python virtual environment (should not be committed to Git)
```

## Architecture and Flow

```
┌────────────┐   POST /chat (JSON)   ┌──────────────┐   CrewAI    ┌────────────────┐
│  Browser   │ ────────────────────▶ │ Flask server │ ──────────▶ │ Ollama (Docker)│
│ index.html │ ◀──────────────────── │   (Python)   │ ◀────────── │ 127.0.0.1:11434│
└────────────┘   { "answer": ... }   └──────────────┘             └────────────────┘
```

Step by step:

1. The user types a message and picks a model on the web page.
2. `script.js` sends `POST /chat` with `message`, `history`, and `model` as JSON.
3. The server calls `ask_ai()` in `general.py`.
4. `ask_ai()` builds an `LLM` pointing at Ollama, then an `Agent` (general assistant and coding helper), a `Task`, and a `Crew`, and runs `kickoff()`.
5. The answer is returned as `{"answer": "..."}` and shown as a bubble in the chat.

## Prerequisites

- Python 3.12 (matches the original virtual environment)
- Docker (to run Ollama), or Ollama installed directly on the machine
- Enough disk space and RAM for the models (`llama3.2:3b` needs about 2 GB, `qwen2.5-coder:1.5b` about 1 GB)

## Installation

### 1. Set up the project and virtual environment

The `.venv` folder in the zip was created on Windows, so it should not be reused on another machine. Create a new one:

```bash
cd "project dev"

# Windows
python -m venv .venv
.venv\Scripts\activate

# macOS / Linux
python3 -m venv .venv
source .venv/bin/activate
```

### 2. Install dependencies

`requirements.txt` currently lists only `flask` and `requests`, but `general.py` also needs `crewai`, so install it as well:

```bash
pip install flask requests crewai
```

> If you use the database example in `app.py`, also install `flask-sqlalchemy`.

### 3. Run Ollama with Docker

```bash
docker run -d \
  --name ollama \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  ollama/ollama
```

### 4. Pull the models

```bash
docker exec -it ollama ollama pull llama3.2:3b
docker exec -it ollama ollama pull qwen2.5-coder:1.5b
```

Check that Ollama is running:

```bash
curl http://127.0.0.1:11434
# Expected output: Ollama is running
```

## Usage

### Terminal mode (works right away)

```bash
python general.py
```

Choose model `1` (Llama 3.2) or `2` (Qwen Coder), then type your questions. Type `exit` or `quit`, or press `Ctrl+C`, to leave.

### Web mode

```bash
python app.py
```

Then open `http://127.0.0.1:5000` in your browser.

> **Note:** Web mode needs the `/chat` endpoint wired up before it fully works. See [Current Status and Recommended Fixes](#current-status-and-recommended-fixes).

## File-by-File Details

### `general.py`

| Part | Purpose |
|------|---------|
| `MODELS` | Dictionary of models for the CLI menu (keys `"1"`, `"2"`) |
| `ask_ai(question, model_name)` | Builds the LLM, Agent, Task, and Crew, sends the question to the model, and returns the answer as text |
| `main()` | Terminal chat loop; handles input, exit commands, and errors |

Settings used in `ask_ai()`:

- **LLM:** `base_url="http://127.0.0.1:11434"`, `temperature=0.5`
- **Agent:** role is *General AI Assistant and Coding Helper*; answers in plain text, avoids unnecessary JSON, and admits when it doesn't know
- **Task:** answers the question directly in the user's language and gives code examples when appropriate
- **Crew:** uses `Process.sequential` with a single agent

### `app.py`

The main Flask file. It currently contains:

- `GET /` — a plain-text "Welcome to My Store" page
- `POST /add-product` — accepts JSON `{name, price}` and saves a product to SQLite (`store.db`)

This is store demo code that is not related to the chat system. See below for details.

### `templates/index.html`

The Thai-language chat page: header (logo, title, clear-chat button), message area, status bar, model selector, and the message input form.

### `static/script.js`

- `addMessage(role, content)` — appends a message bubble (`user` or `assistant`) and auto-scrolls to the bottom
- `submit` handler — sends `fetch("/chat")` as a POST, handles errors, stores `history`, and trims it to the latest 10 messages
- "Clear chat" handler — clears history and shows the welcome screen again

Request format sent:

```json
{
  "message": "the user's message",
  "history": [
    { "role": "user", "content": "..." },
    { "role": "assistant", "content": "..." }
  ],
  "model": "llama3.2:3b"
}
```

Expected response format:

```json
{ "answer": "the AI's answer" }
```

On failure, return a non-2xx HTTP status with `{ "error": "message" }`.

### `static/style.css`

Dark theme in navy and purple tones. User bubbles are right-aligned and AI bubbles are left-aligned. Supports small screens (`max-width: 600px`).

## Supported Models

| Model | Ollama name | Best for |
|-------|-------------|----------|
| Llama 3.2 3B | `llama3.2:3b` | General conversation |
| Qwen 2.5 Coder 1.5B | `qwen2.5-coder:1.5b` | Writing and fixing code |

CrewAI (via LiteLLM) expects model names with an `ollama/` prefix, such as `ollama/llama3.2:3b`. The dropdown values in the web page have no prefix, so the server must add it.

## Customization

**Add a new model**

1. `docker exec -it ollama ollama pull <model-name>`
2. Add it to `MODELS` in `general.py`, e.g. `"3": {"name": "...", "model": "ollama/<model-name>"}`
3. Add an `<option value="<model-name>">` to `templates/index.html`

**Change the AI's personality and behavior** — edit the `Agent`'s `role`, `goal`, and `backstory`, and the `Task`'s `description` in `ask_ai()`.

**Tune answer creativity** — change `temperature` (lower = steadier and more precise, higher = more varied).

**Point to a different Ollama host** — change `base_url` in `ask_ai()`, for example when Ollama runs on another machine.

**Change how much history is kept** — edit `history.slice(-10)` in `script.js`.

## Current Status and Recommended Fixes

Reviewing the code shows the frontend and backend are not fully connected yet. To make web mode work, apply the following:

1. **No `/chat` endpoint** — `script.js` calls `POST /chat`, but `app.py` has no such route.
2. **`app.py` imports a missing module** — `from models.db import db, Product` refers to a `models/` folder that isn't in the project, so running `app.py` raises `ModuleNotFoundError`.
3. **`app.py` doesn't render `index.html`** — the `/` route returns inline HTML instead of calling `render_template("index.html")`.
4. **`requirements.txt` is incomplete** — it is missing `crewai` (and `flask-sqlalchemy` if you use the database).
5. **Conversation history is unused** — the browser sends `history`, but `ask_ai()` only accepts a single question, so the AI has no memory of earlier turns.
6. **`db.create_all()` needs an application context** — in Flask-SQLAlchemy 3.x it must be wrapped in `with app.app_context():`.

### Example `app.py` for chat

The code below replaces the store demo and connects the web page to `general.py`:

```python
from flask import Flask, render_template, request, jsonify
from general import ask_ai

app = Flask(__name__)


@app.route("/")
def index():
    return render_template("index.html")


@app.route("/chat", methods=["POST"])
def chat():
    data = request.get_json(silent=True) or {}
    message = (data.get("message") or "").strip()
    model = (data.get("model") or "llama3.2:3b").strip()

    if not message:
        return jsonify({"error": "Please enter a message"}), 400

    try:
        answer = ask_ai(message, f"ollama/{model}")
        return jsonify({"answer": answer})
    except Exception as error:
        return jsonify({"error": str(error)}), 500


if __name__ == "__main__":
    app.run(debug=True)
```

To give the AI conversational memory, read `history` from the request and append it to the `Task`'s `description` before calling `kickoff()`.

### The store demo

The `Product` model, `/add-product`, and `store.db` are unrelated to the chat system. You can delete them if you don't need them. To keep them, create `models/db.py` that declares `db = SQLAlchemy()` and a `Product` class, and install `flask-sqlalchemy`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `ModuleNotFoundError: No module named 'models'` | `app.py` references a `models/db.py` that doesn't exist | Use the example `app.py` above, or create a `models` module |
| `ModuleNotFoundError: No module named 'crewai'` | Not installed | `pip install crewai` |
| Web page shows an error / 404 on `/chat` | No `/chat` route | Add the endpoint shown above |
| `Connection refused` on port 11434 | Ollama isn't running | Check with `docker ps`, then `docker start ollama` |
| `model not found` | Model hasn't been pulled | `docker exec -it ollama ollama pull <model-name>` |
| Very slow responses | Model running on CPU / not enough RAM | Use a smaller model or run Ollama with a GPU |
| Model name rejected by CrewAI | Missing `ollama/` prefix | Use the format `ollama/model-name:tag` |

## Notes for Committing to Git

Create a `.gitignore` so unnecessary files aren't committed:

```
.venv/
__pycache__/
*.pyc
*.db
instance/
.env
```

# Agentic Chatbot with LangGraph (Human-in-the-loop)

A local Streamlit chatbot that uses a LangGraph workflow, Google Gemini, and a collection of tools for web search, weather, stock quotes, calculations, PDF question answering, and human-approved simulated stock purchases.

## Features

- Conversational AI powered by the `gemini-2.5-flash` model.
- Persistent, multi-conversation chat history backed by LangGraph's SQLite checkpointer.
- Tool-aware responses for:
  - Tavily web search for current information.
  - Mathematical calculations.
  - Current stock quotes from Alpha Vantage.
  - Current weather from OpenWeather.
  - Retrieval-augmented generation (RAG) over an uploaded PDF.
- PDF ingestion using `PyPDFLoader`, recursive text splitting, Google embeddings, and a local FAISS vector store.
- Human-in-the-loop approval for simulated stock purchases. A requested purchase pauses the graph until it is explicitly approved or rejected in the UI.
- Streaming assistant output and tool-status indicators.
- Recovery of saved chats and unresolved purchase approvals after Streamlit reruns, browser refreshes, or switching conversations.

## Architecture

| File | Purpose |
| --- | --- |
| `app_hitl_streamlit.py` | Streamlit user interface, chat threading, PDF upload handling, and approval controls. |
| `agentic_chatbot_hitl_backend.py` | LangGraph workflow, Gemini model, tool definitions, PDF ingestion, FAISS retrieval, and SQLite persistence. |
| `requirements.txt` | Python dependencies required by the application. |

At runtime, the application also creates these local artifacts:

| Path | Purpose |
| --- | --- |
| `chatbot.db` | SQLite checkpoints for LangGraph conversation state and interrupted purchase requests. |
| `faiss_db/` | FAISS index generated from the most recently uploaded PDF. |

## Prerequisites

- Python 3.10 or later.
- A Google AI API key for Gemini chat and embeddings.
- A Tavily API key for web search.
- An OpenWeather API key for weather queries.
- Internet access for Gemini, Tavily, OpenWeather, and Alpha Vantage requests.

The stock quote tool currently contains its Alpha Vantage key in the source code. No additional environment variable is required for that tool as the project is currently written, though moving that key to `.env` is recommended before sharing or deploying the application.

## Setup

1. Clone the repository and open its root directory.

   ```powershell
   git clone <repository-url>
   Set-Location Agentic-Chatbot-using-LangGraph
   ```

2. Create and activate a virtual environment.

   ```powershell
   py -m venv .venv
   .\.venv\Scripts\Activate.ps1
   ```

   On macOS or Linux:

   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   ```

3. Install dependencies.

   ```powershell
   python -m pip install --upgrade pip
   python -m pip install -r requirements.txt
   ```

4. Create a `.env` file in the repository root.

   ```dotenv
   GOOGLE_API_KEY=your_google_ai_api_key
   TAVILY_API_KEY=your_tavily_api_key
   OPENWEATHER_API_KEY=your_openweather_api_key
   ```

   `python-dotenv` loads this file when the backend starts. Keep `.env` out of source control because it contains secrets.

5. Start the Streamlit application.

   ```powershell
   streamlit run app_hitl_streamlit.py
   ```

   Streamlit will print a local URL, usually `http://localhost:8501`. Open that URL in a browser.

## API Keys

| Variable | Used by | How to obtain it |
| --- | --- | --- |
| `GOOGLE_API_KEY` | Gemini chat model and document embeddings | Create an API key in Google AI Studio. |
| `TAVILY_API_KEY` | Tavily web-search tool | Create an API key from Tavily. |
| `OPENWEATHER_API_KEY` | OpenWeather geocoding and current-weather endpoints | Create an API key from OpenWeather. |

The weather tool returns a clear missing-key message when `OPENWEATHER_API_KEY` is not configured. Gemini and Tavily initialization may fail at startup or at first use when their credentials are unavailable.

## Using the Application

### Start or resume a conversation

- Enter a prompt in the chat input and submit it.
- Use **New Chat** in the sidebar to create a separate conversation.
- Select a thread ID in the sidebar to load its saved messages.
- Conversation state is written to `chatbot.db`, so chats remain available when the Streamlit server restarts on the same machine.

### Ask general questions

Ask ordinary questions directly. The model answers without calling a tool when no external information or calculation is necessary.

### Search the web

Ask about current events or recent information, for example:

```text
What were the latest developments in renewable energy this week?
```

The agent selects Tavily search when it needs up-to-date web information.

### Calculate a value

Ask for a calculation, for example:

```text
What is math.sqrt(144) + 18 * 4?
```

The calculator supports expressions that use `math` plus `abs`, `round`, `min`, `max`, and `sum`.

### Check weather or a stock quote

Examples:

```text
What is the current weather in Seattle, US?
What is the current price of AAPL?
```

Weather results use metric units: degrees Celsius, meters per second, and kilometers.

### Upload and query a PDF

1. Use the attachment control in the chat input to select a PDF.
2. Optionally include a question in the same submission or send one after upload.
3. Wait for the success toast confirming that the PDF was processed.
4. Ask questions about the PDF content.

The uploaded PDF is saved temporarily only for ingestion and then deleted. Its chunk embeddings are saved in `faiss_db/`. The current implementation maintains one shared local FAISS index, so uploading another PDF replaces the prior index. PDF retrieval is not scoped per conversation or user.

### Approve or reject a stock purchase

Ask the agent to purchase a stock, such as:

```text
Buy 10 shares of MSFT.
```

The agent invokes the purchase tool, pauses the LangGraph workflow, and shows **Approve Purchase** and **Reject Purchase** controls. The chat input is disabled while a decision is pending. Selecting either option resumes the exact interrupted conversation and returns a simulated outcome; this project does not place real trades.

## Development Notes

- Run the app from the repository root. The backend uses relative paths for `chatbot.db` and `faiss_db`.
- The application uses SQLite with `check_same_thread=False` because Streamlit reruns execute in a threaded environment.
- `FAISS.load_local(..., allow_dangerous_deserialization=True)` loads the locally generated index. Do not replace `faiss_db/` with an index from an untrusted source.
- The calculator evaluates a restricted expression environment. It is intended for simple calculations only.
- External service quotas, rate limits, key activation delays, and network failures can affect tool responses.

## Reset Local Data

Stop Streamlit before deleting generated state. To remove all chat history and the current document index, delete these paths from the repository root:

```powershell
Remove-Item chatbot.db -ErrorAction SilentlyContinue
Remove-Item faiss_db -Recurse -Force -ErrorAction SilentlyContinue
```

The next run creates a new database. Upload a PDF again before asking document-specific questions.

## Troubleshooting

| Problem | Likely cause | Resolution |
| --- | --- | --- |
| `streamlit` is not recognized | Virtual environment is inactive or dependencies are not installed. | Activate `.venv` and run `python -m pip install -r requirements.txt`. |
| Gemini authentication error | `GOOGLE_API_KEY` is absent, invalid, or not readable from `.env`. | Verify `.env` is in the repository root, restart Streamlit, and use a valid Google AI key. |
| Search tool fails | Tavily key is absent or invalid. | Set `TAVILY_API_KEY` and restart the app. |
| Weather says the API key is missing or invalid | OpenWeather key is absent, inactive, or invalid. | Set `OPENWEATHER_API_KEY`; newly created keys can take time to activate. |
| PDF question fails before a PDF has been uploaded | No local FAISS index exists. | Attach and submit a PDF, wait for processing to finish, then ask the question again. |
| A prior PDF is no longer available | Another PDF was uploaded and replaced the shared index. | Upload the desired PDF again. |
| Port 8501 is busy | Another Streamlit process is running. | Stop that process or start on another port with `streamlit run app_hitl_streamlit.py --server.port 8502`. |

## Security and Deployment

This repository is intended for local development and demonstration. Before exposing it to other users or deploying it:

- Remove the hard-coded Alpha Vantage key and store it in a secret manager or environment variable.
- Add `.env`, `chatbot.db`, and `faiss_db/` to `.gitignore` if they are not already excluded.
- Replace the shared local SQLite and FAISS storage with isolated, durable storage suitable for multiple users.
- Restrict uploads, validate file sizes and content, and add authentication and authorization.
- Review the tool set and apply rate limits, monitoring, and appropriate API-key management.
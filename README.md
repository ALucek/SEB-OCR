# seb-ocr

<img src="./seb_ocr_logo.png" width=600>

A vision-language pipeline for turning scanned historical pages into structured datasets.  At its core the project leverages Google's [Gemini](https://ai.google.dev/gemini-api/docs/models) family for both vision-OCR and text entity extraction, built specifically for ongoing political science research.

**How it works**

1. **OCR pass** ‑ Each input image is sent to a multimodal model with a handcrafted transcription prompt.  Calls are executed in parallel but throttled by an adaptive rate-limiter so you stay within API quotas.
2. **Sliding-window parsing** ‑ Transcriptions are processed in overlapping windows (size&nbsp;=`WINDOW_SIZE`, step&nbsp;=`WINDOW_STEP`) enabling the LLM to capture cross-page context. The model then returns JSON conforming to a strict Pydantic schema.
3. **Incremental caching** ‑ Window results are cached under `output/window_outputs/` so subsequent runs resume instantly.
4. **Semantic deduplication** ‑ Candidate entities are embedded and clustered by cosine similarity to merge duplicates that appear on multiple pages.
5. **Exports** ‑ Final, de-duplicated entries are written to `entities.json` and a human-friendly `entities.csv`.

The entire flow is a single `main.py` script; no external services or databases required.

## Usage

1. Clone the repository:
    ```bash
    git clone https://github.com/ALucek/seb-ocr.git
    cd seb-ocr
    ```

2. Install dependencies using [uv](https://docs.astral.sh/uv/):

    ```bash
    uv sync
    ```

3.  Create a `.env` file with the required credentials and (optional) tuning knobs.

    ```ini
    # Required
    GEMINI_API_KEY="your-secret-key"

    # Optional overrides
    GEMINI_MODEL="gemini-2.5-flash"               # generation model
    GEMINI_EMBEDDING_MODEL="text-embedding-004"   # embedding model

    # Rate-limiting (set to 0 to disable)
    GEMINI_RPM_LIMIT=50            # requests / minute for generation
    GEMINI_EMBEDDING_RPM_LIMIT=1000 # requests / minute for embeddings

    # Performance tuning
    MAX_WORKERS=10   # parallel threads
    WINDOW_SIZE=5    # pages per sliding window
    WINDOW_STEP=2    # window step size
    ```

4.  Drop page-scanned images in the `input_images/` directory (created automatically on first run).

    • **File-name convention** – Each image must contain its page number so the pipeline can sort pages correctly (e.g. `001.jpg`, `page_2.png`, `3.webp`).  The number is extracted with a simple regex, so any contiguous digits will work.

5.  Run the pipeline (pick a mode):

    ```bash
    # Full end-to-end run (default if omitted)
    uv run main.py all

    # OCR only – produce transcriptions
    uv run main.py transcribe

    # Entity extraction only – requires existing transcriptions
    uv run main.py extract
    ```

    The pipeline creates an `output/` directory:

    * `output/transcriptions/` – one `.txt` file per input image (raw OCR transcription)
    * `output/window_outputs/` – intermediate JSON for each sliding-window entity extraction (cached for incremental runs)
    * `output/final_outputs/`  – `entities.json` and `entities.csv` containing the deduplicated entity list.
    
    
Any errors are logged to the console; retries are handled automatically.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) for details.

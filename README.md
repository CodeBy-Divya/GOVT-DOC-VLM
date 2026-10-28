<div align="center">

# 🏛️ GOVT-DOC-VLM

### Local, privacy-first document extraction for scanned government PDFs — powered by Qwen 3.5 Vision-Language Models

![Python](https://img.shields.io/badge/Python-3.10+-blue?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/Backend-FastAPI-009688?logo=fastapi&logoColor=white)
![React](https://img.shields.io/badge/Frontend-React%20%2B%20Vite-61DAFB?logo=react&logoColor=black)
![Model](https://img.shields.io/badge/Model-Qwen%203.5%20VLM-purple)
![Privacy](https://img.shields.io/badge/Runs-100%25%20Local-success)

**Built by [Divya Singh](https://github.com/CodeBy-Divya)**

</div>

---

## 📖 About the Project

Government paperwork in India (certificates, forms, orders, notifications, records) mostly lives as **scanned PDFs**. They have no selectable text, layouts vary wildly, and stamps, tables, signatures and regional-language content make classic OCR unreliable.

I built **GOVT-DOC-VLM** to tackle this with a Vision-Language Model instead of traditional OCR. The model *looks* at each page the way a person does, so it understands layout, tables and context, and it runs **entirely on local hardware**, so sensitive documents never touch a third-party API.

```
Scanned PDF  →  Page images  →  Qwen 3.5 VLM  →  Structured text
```

### ✨ Features

- 📄 **PDF → image pipeline** with configurable DPI (PyMuPDF)
- 🧠 **VLM-based extraction** that handles layout, tables and mixed content better than plain OCR
- 🔒 **Fully offline at runtime**: no cloud APIs, no API keys, no data leaving your machine
- ⚖️ **Two model sizes**: 27B (FP8) for a single 16 GB GPU, 122B-A10B for high-memory setups
- 🖥️ **Side-by-side UI**: original page image next to the extracted text
- 🧪 **Mock mode** to build and test the whole app on a laptop without a GPU
- 🎯 **Fine-tuning workspace** for adapting the model to government document types

---

## 🏗️ Architecture

```
┌────────────┐   upload PDF   ┌──────────────┐   page images   ┌────────────────────┐
│ React UI   │ ─────────────► │ FastAPI      │ ──────────────► │ Local Qwen 3.5     │
│ (Vite)     │ ◄───────────── │ backend      │ ◄────────────── │ model server :8000 │
└────────────┘  images + text └──────────────┘   extracted text└────────────────────┘
                                    │
                                    ├── pdf_processor.py  (PDF → images)
                                    ├── model_client.py   (HTTP call to local model)
                                    └── mock_client.py    (fake output for testing)
```

| Component | Role |
|---|---|
| `main.py` | FastAPI app: receives PDFs, runs the pipeline, returns images + text |
| `pdf_processor.py` | Renders each PDF page to an image |
| `model_client.py` | Sends images to the local Qwen server over plain HTTP (`httpx`) |
| `mock_client.py` | Returns placeholder text for GPU-free testing |
| `config.py` | Model name, ports, DPI and the `USE_MOCK` switch |
| `start_model.sh` | Starts the local Qwen model server (downloads weights on first run) |

---

## 📂 Project Structure

```
govt-doc-vlm/
├── doc-qwen3.5-27b/              # 27B version, fits in 16GB VRAM cleanly
│   ├── backend/
│   │   ├── config.py             # the only file that differs between the two folders
│   │   ├── main.py
│   │   ├── pdf_processor.py
│   │   ├── model_client.py
│   │   ├── mock_client.py
│   │   └── requirements.txt
│   ├── frontend/                 # React + Vite app
│   └── start_model.sh
│
├── doc-qwen3.5-122b-a10b/        # 122B version, VRAM + shared RAM
│   ├── backend/
│   ├── frontend/
│   └── start_model.sh
│
└── finetune/                     # fine-tuning scripts and data
```

### Choosing a model

| | **27B** | **122B-A10B** |
|---|---|---|
| Model ID | `Qwen/Qwen3.5-27B-FP8` | `Qwen/Qwen3.5-122B-A10B` |
| Backend port | `8001` | `8002` |
| Approx. size | ~14 GB | ~65 GB |
| Memory | Fits fully in 16 GB VRAM | Split across VRAM + shared RAM (`device_map=auto`) |
| Best for | Speed on standard workstations | Maximum accuracy on difficult documents |

**Reference hardware (NVIDIA A4000):** 16 GB VRAM + 86 GB shared RAM, about 102 GB effective.

---

## 🚀 Getting Started

### Prerequisites

- Python 3.10+
- Node.js 18+ and npm
- *(Real model only)* NVIDIA GPU with CUDA 12.4 drivers

### Step 1: Run on a laptop (mock mode, no GPU)

```bash
# Backend
cd doc-qwen3.5-27b/backend
python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt
python -m uvicorn main:app --port 8001 --reload

# Frontend (new terminal)
cd doc-qwen3.5-27b/frontend
npm install
npm run dev
```

> ⚠️ Use `python -m uvicorn`, not plain `uvicorn`. Plain `uvicorn` may pick up the system Python and miss your venv packages.

Open **http://localhost:5173**, upload a PDF, and you will see the real page images with mock text below.

### Step 2: Run with the real model (GPU machine)

```bash
# 1. Install GPU libraries
pip install torch --index-url https://download.pytorch.org/whl/cu124
pip install "transformers[serving] @ git+https://github.com/huggingface/transformers.git@main"
pip install torchvision accelerate huggingface_hub

# 2. Start the model server (downloads weights on first run)
cd doc-qwen3.5-27b
bash start_model.sh

# 3. In backend/config.py set:  USE_MOCK = False

# 4. Start the backend
cd backend
python -m uvicorn main:app --port 8001 --reload

# 5. Start the frontend
cd ../frontend
npm install && npm run dev
```

### Optional: pre-download the models

```bash
huggingface-cli download Qwen/Qwen3.5-27B-FP8
huggingface-cli download Qwen/Qwen3.5-122B-A10B
```

Weights are cached in `~/.cache/huggingface/hub/` (about 14 GB for 27B, 65 GB for 122B) and reused on every later run.

---

## 🔐 Privacy by Design

```
React UI → FastAPI backend → httpx POST → localhost:8000 → Qwen model on GPU
```

- No `openai` package and no API keys
- No internet needed at runtime after the one-time model download
- Every document is processed on your own hardware

---

## ⚙️ Configuration

Set in `backend/config.py`:

| Setting | Description |
|---|---|
| `MODEL_NAME` | Hugging Face model ID served by `start_model.sh` |
| `BACKEND_PORT` | Port reference (8001 for 27B, 8002 for 122B) |
| `USE_MOCK` | `True` for fake output (laptop), `False` for the real model (GPU) |
| `PDF_DPI` | Page render resolution (default 200; use 150 if you hit timeouts) |

---

## 🛠️ Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `No module named 'pymupdf'` / `'fitz'` | Ran `uvicorn` instead of `python -m uvicorn` | Use `python -m uvicorn main:app --port 8001 --reload` |
| Images not showing | PyMuPDF import failing silently | Same fix |
| `ModuleNotFoundError` | Wrong Python or venv not active | `source venv/bin/activate` |
| `[CONNECTION ERROR]` in output | Model server not running | Run `bash start_model.sh` first |
| `[TIMEOUT]` in output | Page image too large | Lower `PDF_DPI` from 200 to 150 |

---

## 🗺️ Roadmap

- [ ] Structured JSON output (key-value fields per document type)
- [ ] Hindi and regional-language support
- [ ] Batch processing of multiple PDFs
- [ ] Export to DOCX / CSV / JSON
- [ ] Fine-tuned checkpoints for specific document categories
- [ ] Docker setup for one-command deployment

---

## 👩‍💻 Author

**Divya Singh**
B.Tech Computer Science, IIT Patna · AI / ML Engineer · Data Analyst · Developer

- GitHub: [@CodeBy-Divya](https://github.com/CodeBy-Divya)
- LinkedIn: _add your link here_
- Email: _add your email here_

## 🙏 Credits

This project started from the open-source [GOVT-DOC-VLM](https://github.com/Maddy-V1/GOVT-DOC-VLM) by Maddy-V1, which I forked and continue to develop. It builds on [Qwen](https://github.com/QwenLM), [FastAPI](https://fastapi.tiangolo.com/), [PyMuPDF](https://pymupdf.readthedocs.io/), [Hugging Face Transformers](https://github.com/huggingface/transformers), [React](https://react.dev/) and [Vite](https://vitejs.dev/).

## 📄 License

Add a `LICENSE` file (for example MIT). If the original repo has a license, keep it compatible with that one.

---

<div align="center">⭐ If this project helps you, consider giving it a star!</div>

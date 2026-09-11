<h1 align="center">tewtr</h1>

<p align="center">
  <em>turns a PDF into a tutor that has actually read it</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/python-0b0b0b?style=flat-square&logo=python&logoColor=white" alt="python">
  <img src="https://img.shields.io/badge/fastapi-0b0b0b?style=flat-square&logo=fastapi&logoColor=white" alt="fastapi">
  <img src="https://img.shields.io/badge/react-0b0b0b?style=flat-square&logo=react&logoColor=white" alt="react">
  <img src="https://img.shields.io/badge/typescript-0b0b0b?style=flat-square&logo=typescript&logoColor=white" alt="typescript">
</p>

---

## the problem

Feeding a whole textbook to a language model does not work. The context window runs out,
and even when it does not, quality falls apart somewhere in the middle of a long document.
Splitting it into independent pages fixes the context problem and creates a worse one: page 40
no longer knows that page 39 introduced the variable it is using.

tewtr processes one page at a time but carries a running summary forward, so each page is
explained with knowledge of what came before it.

## how it works

Two models, two different jobs:

```
PDF page
   ↓
  render to image
   ↓
  Llama 4 Maverick (vision)        reads the page: text, diagrams, equations, layout
   ↓
  transcription  +  previous_context
   ↓
  GPT-OSS-120B (language)          explains it, builds flashcards and questions
   ↓
  structured JSON  →  becomes previous_context for the next page
```

The vision model does not try to teach, and the language model never sees the raw image.
Each stage does the thing it is good at, which keeps both prompts short and the output
predictable.

Both run on **SambaNova**, which serves them fast enough that a page finishes in seconds
rather than minutes. Model choice and provider are set in one place, so swapping either is a
one-line change:

```python
self.vlm = VLMModel(provider="sambanova", clarifai_model_id="Llama-4-Maverick-17B-128E-Instruct")
```

`VLMModel` also speaks to Clarifai and Baseten behind the same OpenAI-shaped interface, so a
provider outage or a price change does not mean rewriting the pipeline.

## what it produces

For every page, structured JSON rather than a wall of prose:

- a transcription of what is actually on the page
- an explanation written for someone learning it for the first time
- flashcards
- questions to check understanding

The frontend turns that into a notebook you can read, and a chat box that answers questions
against the document rather than against the model's general knowledge.

## running it

Backend:

```bash
cd backend
pip install -r requirements.txt
uvicorn app.main:app --reload
```

You need a `.env` with a key for whichever provider you point it at:

```
SAMBANOVA_API_KEY=...
CLARIFAI_API_KEY=...      # optional
NEMOTRON_API_KEY=...      # optional, Baseten
```

Frontend:

```bash
cd ai_tutor
npm install
npm run dev
```

## layout

```
backend/app/
  main.py            FastAPI routes, page-by-page orchestration
  agent.py           VLMAgent: transcribe → explain → flashcards → chat
  model.py           provider abstraction (SambaNova, Clarifai, Baseten)
  pdf_processor.py   PDF to page images
ai_tutor/            React + TypeScript + Tailwind frontend
```

## honest notes

The context carried between pages is a summary, not the full history, so a reference to
something forty pages back can still be missed. Pages are processed in order because each one
depends on the last, which means throughput is bounded by the slowest page rather than by how
many pages you can run at once. Both are fixable and neither is fixed yet.

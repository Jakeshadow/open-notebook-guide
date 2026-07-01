# Chapter 7: API-Driven Workflows and Automation

**Estimated reading time:** 16 minutes
**What you'll build:** Full API automation — batch document ingestion, automated podcast generation, Python client patterns, webhook integration, and n8n/Zapier recipes.
**Prerequisites:** Chapter 1 — working deployment with API port 5055 accessible.

---

Open Notebook has a full REST API on port 5055. Everything you can do in the UI, you can do programmatically. This chapter turns your instance from a web app into an automation engine.

> **Pain Point #7:** One of the main reasons to self-host instead of using NotebookLM is the API. NotebookLM has no API. Open Notebook's API enables batch processing thousands of documents, building custom frontends, and integrating document AI into existing workflows.

## API Reference (Concise)

- **Base URL:** `https://notebook.yourdomain.com/api`
- **Auth:** `Authorization: Bearer your-secret-key` or `X-API-Key: your-secret-key`

### Health Check

```bash
curl -H "Authorization: Bearer $API_KEY" https://notebook.yourdomain.com/api/health
```

### Create Notebook

```bash
curl -X POST https://notebook.yourdomain.com/api/notebooks \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title": "Q3 Research","description": "Market analysis"}'
```

### Upload PDF

```bash
curl -X POST https://notebook.yourdomain.com/api/sources \
  -H "Authorization: Bearer $API_KEY" \
  -F "file=@document.pdf" -F "notebook_id={notebook_id}"
```

### Add URL/YouTube

```bash
curl -X POST https://notebook.yourdomain.com/api/sources \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com/article","notebook_id": "{id}"}'
```

### Chat

```bash
curl -X POST https://notebook.yourdomain.com/api/chat \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"notebook_id": "{id}","message": "Summarize key findings","stream": false}'
```

### Generate Podcast

```bash
curl -X POST https://notebook.yourdomain.com/api/podcast/generate \
  -H "Authorization: Bearer $API_KEY" \
  -d '{"notebook_id": "{id}","speaker_count": 2,"style": "conversational"}'
```

### Download Podcast

```bash
curl https://notebook.yourdomain.com/api/podcast/download/{podcast_id} \
  -H "Authorization: Bearer $API_KEY" -o podcast.mp3
```

## Python Client: Batch Document Ingestion

```python
#!/usr/bin/env python3
import os, time, requests
from pathlib import Path

API_URL = "https://notebook.yourdomain.com/api"
API_KEY = os.environ["OPEN_NOTEBOOK_API_KEY"]
NOTEBOOK_ID = "your-notebook-id"
PDF_DIR = "/path/to/pdfs"

def upload_pdf(filepath):
    with open(filepath, "rb") as f:
        resp = requests.post(f"{API_URL}/sources",
            headers={"Authorization": f"Bearer {API_KEY}"},
            files={"file": (os.path.basename(filepath), f, "application/pdf")},
            data={"notebook_id": NOTEBOOK_ID})
    return resp.json()["id"]

def wait_for_processing(source_id, timeout=300):
    start = time.time()
    while time.time() - start < timeout:
        status = requests.get(f"{API_URL}/sources/{source_id}",
            headers={"Authorization": f"Bearer {API_KEY}"}).json().get("status")
        if status == "completed": return "completed"
        if status == "error": return "error"
        time.sleep(5)
    return "timeout"

pdf_files = list(Path(PDF_DIR).glob("*.pdf"))
for i, pdf in enumerate(pdf_files):
    print(f"[{i+1}/{len(pdf_files)}] {pdf.name}...", end=" ")
    source_id = upload_pdf(str(pdf))
    status = wait_for_processing(source_id)
    print("✓" if status == "completed" else f"✗ {status}")
    time.sleep(5)
```

## Python Client: Automated Podcast Pipeline

```python
import os, time, requests
from datetime import datetime

API_URL = "https://notebook.yourdomain.com/api"
API_KEY = os.environ["OPEN_NOTEBOOK_API_KEY"]

def get_source_count(notebook_id):
    resp = requests.get(f"{API_URL}/sources?notebook_id={notebook_id}",
        headers={"Authorization": f"Bearer {API_KEY}"})
    return len(resp.json().get("sources", []))

def generate_podcast(notebook_id):
    return requests.post(f"{API_URL}/podcast/generate",
        headers={"Authorization": f"Bearer {API_KEY}"},
        json={"notebook_id": notebook_id, "speaker_count": 2}).json()["podcast_id"]

def wait_for_podcast(podcast_id, timeout=600):
    start = time.time()
    while time.time() - start < timeout:
        resp = requests.get(f"{API_URL}/podcast/status/{podcast_id}",
            headers={"Authorization": f"Bearer {API_KEY}"})
        status = resp.json().get("status")
        if status == "completed":
            return resp.json()["download_url"]
        time.sleep(10)
    raise TimeoutError()

def download(url, path):
    with requests.get(url, stream=True) as r, open(path, "wb") as f:
        for chunk in r.iter_content(8192): f.write(chunk)

if get_source_count("notebook-id") >= 5:
    pid = generate_podcast("notebook-id")
    url = wait_for_podcast(pid)
    download(url, f"podcast-{datetime.now():%Y%m%d-%H%M}.mp3")
```

## Minimal Custom Frontend (Flask)

```python
from flask import Flask, request, render_template_string
import requests, os

API_URL = "https://notebook.yourdomain.com/api"
API_KEY = os.environ["OPEN_NOTEBOOK_API_KEY"]

HTML = """<!DOCTYPE html><html><head><title>DocQ</title></head>
<body style="max-width:700px;margin:40px auto;font-family:system-ui">
<h1>DocQ</h1>
<form method="post">
  <textarea name="query" rows="3" style="width:100%">{{query}}</textarea>
  <button type="submit">Ask</button>
</form>
{% if answer %}<div style="margin-top:20px;background:#f5f5f5;padding:15px">{{answer}}</div>{% endif %}
</body></html>"""

app = Flask(__name__)

@app.route("/", methods=["GET", "POST"])
def home():
    answer, query = "", ""
    if request.method == "POST":
        query = request.form["query"]
        resp = requests.post(f"{API_URL}/chat",
            headers={"Authorization": f"Bearer {API_KEY}"},
            json={"notebook_id": "your-id", "message": query, "stream": False})
        answer = resp.json().get("response", "Error")
    return render_template_string(HTML, query=query, answer=answer)

if __name__ == "__main__":
    app.run(port=5000)
```

## n8n Automation Recipes

- **RSS → Summarize → Slack:** RSS Feed Read → POST /api/sources → Wait → POST /api/chat → Slack Message
- **Email PDF → Auto-Ingest:** Gmail Trigger (PDF attachment) → Download → POST /api/sources → Reply
- **Daily Podcast Newsletter:** Schedule (7 AM) → POST /api/podcast/generate → Wait → GET download → Email MP3

## Zapier Recipes

| Trigger               | Action                                         |
| --------------------- | ---------------------------------------------- |
| New Google Drive file | POST /api/sources (upload)                     |
| New Airtable row      | POST /api/sources (URL)                        |
| RSS new item          | POST /api/sources → POST /api/chat (summarize) |

---

Next: Chapter 8 — Monitoring, Alerting, and Maintenance →

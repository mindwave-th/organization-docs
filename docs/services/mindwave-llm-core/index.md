# Mindwave Local LLM

ระบบ Local LLM แบบครบวงจร — รันโมเดล Hugging Face บนเครื่องตัวเอง (Metal GPU บน Mac / CUDA / CPU)
พร้อมระบบจัดการโมเดล, streaming (SSE), thinking mode และ settings แบบละเอียดที่เก็บเป็นไฟล์ JSON สำหรับ AI core

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────┐
│  Clients (curl, App, Browser)                    │
└──────────────────────┬───────────────────────────┘
                       │ HTTP :8000
┌──────────────────────▼───────────────────────────┐
│  api/server.py — FastAPI                         │
│  chat · generate · models · settings · SSE       │
└──────────────────────┬───────────────────────────┘
┌──────────────────────▼───────────────────────────┐
│  core/ — AI Core                                 │
│  engine.py        inference + stream + thinking  │
│  model_manager.py download/load/unload/delete    │
│  config.py        persistent settings (JSON)     │
└──────────────────────┬───────────────────────────┘
                       │
        models/  (โมเดลที่ดาวน์โหลดไว้)
        core/settings/settings.json  (การตั้งค่า)
```

## 🚀 Quick Start

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt

# start server (host/port อ่านจาก settings)
.venv/bin/python main.py
```

ดาวน์โหลด + โหลดโมเดล:

```bash
# ดาวน์โหลด (ทำงาน background — เช็กสถานะได้)
curl -X POST localhost:8000/v1/models/download \
  -H 'Content-Type: application/json' \
  -d '{"model": "google/gemma-3-12b-it"}'

curl localhost:8000/v1/models/download/status

# โหลดเข้า memory (ถ้ายังไม่ดาวน์โหลด จะดาวน์โหลดให้เอง)
curl -X POST localhost:8000/v1/models/load \
  -H 'Content-Type: application/json' \
  -d '{"model": "google/gemma-3-12b-it"}'
```

แชต:

```bash
# non-stream
curl -X POST localhost:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"messages":[{"role":"user","content":"Hello!"}]}'

# stream (SSE, OpenAI-compatible chunks)
curl -N -X POST localhost:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"messages":[{"role":"user","content":"Hello!"}],"stream":true}'

# เปิด thinking เฉพาะ request นี้
curl -X POST localhost:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"messages":[{"role":"user","content":"แก้สมการ x^2=4"}],"thinking":true}'
```

## 📡 API Reference

| Method | Path | คำอธิบาย |
|---|---|---|
| POST | `/v1/chat/completions` | แชต OpenAI-compatible (`stream`, `thinking`, override params ได้) |
| POST | `/v1/generate` | text-in/text-out แบบง่าย (`prompt`, `system`, `stream`) |
| GET | `/v1/models` | โมเดลในเครื่อง + สถานะโหลด |
| POST | `/v1/models/download` | ดาวน์โหลดโมเดลจาก HF Hub (background) |
| GET | `/v1/models/download/status` | สถานะการดาวน์โหลด |
| POST | `/v1/models/load` | โหลดโมเดลเข้า memory |
| POST | `/v1/models/unload` | ปลดโมเดลออกจาก memory |
| DELETE | `/v1/models/{model_id}` | ลบโมเดลออกจาก disk |
| GET | `/v1/providers` | รายชื่อ provider (local + remote, mask API key) |
| GET | `/v1/providers/{name}/models` | โมเดลฝั่ง remote provider |
| GET | `/v1/settings` | อ่าน settings ทั้งหมด + path ไฟล์ |
| PATCH | `/v1/settings` | อัปเดตบางส่วน (deep-merge แล้วบันทึกลงไฟล์) |
| POST | `/v1/settings/reset` | คืนค่า default |
| GET | `/health` | health check |
| GET | `/docs` | Swagger UI |

Override พารามิเตอร์ต่อ request ได้ใน chat/generate: `temperature`, `top_k`, `top_p`,
`max_tokens`, `repetition_penalty`, `do_sample`, `thinking`, `model`, `provider`

## 🌐 Remote Providers

นอกจาก local แล้ว ต่อ API ภายนอกที่เป็น OpenAI-compatible ได้ทุกเจ้า
(OpenAI, OpenRouter, Groq, Together, vLLM, LM Studio, Ollama `/v1`)
ตั้งค่าใน `settings.providers.remote`:

```bash
# เพิ่ม/แก้ provider
curl -X PATCH localhost:8000/v1/settings -H 'Content-Type: application/json' -d '{
  "providers": {"remote": {"groq": {
    "type": "openai-compatible",
    "base_url": "https://api.groq.com/openai/v1",
    "api_key_env": "GROQ_API_KEY",
    "default_model": "llama-3.3-70b-versatile",
    "timeout": 120,
    "enabled": true
  }}}}'

# เรียกผ่าน provider นั้น (stream ได้เหมือน local ทุกอย่าง)
curl -X POST localhost:8000/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"messages":[{"role":"user","content":"hi"}],"provider":"groq","stream":true}'
```

- `providers.default` — provider ที่ใช้เมื่อ request ไม่ระบุ (`"local"` = engine ในเครื่อง)
- API key ใส่ได้ 2 ทาง: `api_key_env` (อ่านจาก env — แนะนำ) หรือ `api_key` ในไฟล์ settings ตรงๆ
- `reasoning_content` จาก provider ที่รองรับ thinking จะถูกส่งต่อเป็น field เดียวกับ local
- มี `openai` กับ `ollama` ตั้งต้นไว้ให้แล้ว (ปิดอยู่ — เปิดด้วย `"enabled": true`)

## ⚙️ Settings (AI Core)

เก็บที่ `core/settings/settings.json` (เปลี่ยน path ด้วย env `MINDWAVE_SETTINGS_PATH`)
แก้ผ่าน API หรือแก้ไฟล์ตรงๆ ก็ได้ — โครงสร้าง:

```json
{
  "model": {
    "model_dir": "models",
    "default_model": "google/gemma-3-12b-it",
    "device": "auto",
    "dtype": "auto",
    "trust_remote_code": false,
    "load_on_startup": false
  },
  "generation": {
    "max_new_tokens": 1024,
    "temperature": 0.7,
    "top_k": 50,
    "top_p": 0.95,
    "repetition_penalty": 1.1,
    "do_sample": true,
    "stop": [],
    "seed": null
  },
  "thinking": {
    "enabled": false,
    "tag_open": "<think>",
    "tag_close": "</think>",
    "include_in_response": false
  },
  "streaming": { "enabled": true, "stream_timeout": 600 },
  "server": { "host": "0.0.0.0", "port": 8000, "log_level": "info" }
}
```

ตัวอย่างอัปเดตผ่าน API:

```bash
curl -X PATCH localhost:8000/v1/settings \
  -H 'Content-Type: application/json' \
  -d '{"thinking":{"enabled":true,"include_in_response":true},"generation":{"temperature":0.4}}'
```

### Thinking mode

- `thinking.enabled` — ขอ reasoning จาก chat template (โมเดลที่รองรับ `enable_thinking` เช่น Qwen; โมเดลที่ไม่รองรับจะ fallback อัตโนมัติ)
- `include_in_response: false` — ตัด `<think>...</think>` ออกจากคำตอบ
- `include_in_response: true` — ส่ง reasoning กลับมาด้วย (non-stream: ฟิลด์ `reasoning_content` / stream: delta `reasoning_content`)

## 🗂️ โครงสร้างไฟล์

```
mindwave-gemma-ai/
├── main.py                  # entrypoint — start uvicorn จาก settings
├── api/
│   ├── server.py            # FastAPI routes
│   └── schemas.py           # Pydantic schemas
├── core/
│   ├── config.py            # SettingsManager (persist JSON)
│   ├── model_manager.py     # download/load/unload/delete โมเดล
│   ├── engine.py            # inference + streaming + thinking filter
│   └── settings/settings.json
├── models/                  # โมเดลที่ดาวน์โหลด (gitignored)
└── requirements.txt
```

## 💡 หมายเหตุ

- โมเดล Gemma บน HF Hub ต้องยอมรับ license ก่อน → `huggingface-cli login` แล้วกด accept ที่หน้าโมเดล
- `device: auto` ใช้ `device_map="auto"` ของ accelerate — Mac จะใช้ MPS (Metal), เครื่อง NVIDIA ใช้ CUDA
- generate ทีละ request (lock) กัน memory เต็มจากการรันพร้อมกัน
- ระบบ Ollama proxy เวอร์ชันเก่าถูกแทนที่แล้ว — ดูได้จาก git history (`git show 17bbbe9`)

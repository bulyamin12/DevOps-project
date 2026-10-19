# Project Dev — Local AI Chat & Agent

โปรเจกต์แชท AI ที่รันโมเดลบนเครื่องตัวเอง 100% ผ่าน **Ollama (รันใน Docker)** โดยมีทั้ง **หน้าเว็บแชท (Flask)** และ **โหมดเทอร์มินัลที่ใช้เอเจนต์ของ CrewAI**

## ความสามารถหลัก

- แชทกับโมเดล AI ผ่านหน้าเว็บ (Flask + HTML/CSS/JS)
- เลือกโมเดลได้หลายตัว (Llama 3.2 3B, Qwen2.5-Coder 1.5B)
- โหมดเทอร์มินัล พิมพ์ถาม-ตอบกับเอเจนต์ได้ทันที พิมพ์ `exit` เพื่อออก
- ใช้ **CrewAI** สร้างเอเจนต์ผู้ช่วยทั่วไปและช่วยเขียนโค้ด
- ข้อมูลไม่ออกนอกเครื่อง ไม่ต้องใช้ API key และไม่มีค่าใช้จ่ายรายครั้ง
- รองรับภาษาไทย

## สถาปัตยกรรม

```
┌──────────────┐     ┌──────────────┐     ┌────────────────────────┐     ┌──────────┐
│ หน้าเว็บ     │ ──► │ Flask        │ ──► │ Ollama (Docker)        │ ──► │ โมเดล AI │
│ index.html   │     │ app.py       │     │ 127.0.0.1:11434        │     │          │
└──────────────┘     └──────────────┘     └────────────────────────┘     └──────────┘

┌──────────────┐     ┌──────────────┐     ┌────────────────────────┐
│ เทอร์มินัล    │ ──► │ general.py   │ ──► │ Ollama (Docker)        │
│              │     │ (CrewAI)     │     │                        │
└──────────────┘     └──────────────┘     └────────────────────────┘
```

- **Flask และ Python** รันบนเครื่องโดยตรง (ใน virtual environment `.venv`)
- **Ollama** รันใน Docker container และเปิด port `11434` ให้โค้ดเรียกใช้

## โครงสร้างโปรเจกต์

```
project dev/
├── .venv/              # Python virtual environment
├── static/
│   ├── script.js       # ตรรกะฝั่งหน้าเว็บ (ส่งข้อความ, แสดงคำตอบ)
│   └── style.css       # สไตล์ของหน้าเว็บ
├── templates/
│   └── index.html      # หน้าแชทหลัก
├── app.py              # เว็บเซิร์ฟเวอร์ Flask
├── general.py          # เอเจนต์ CrewAI + โหมดแชทในเทอร์มินัล
├── config.yaml         # ไฟล์ตั้งค่า
├── requirements.txt    # รายการไลบรารี Python
└── README.md
```

## โมเดลที่รองรับ

| รหัส | ชื่อ | โมเดลใน Ollama | เหมาะกับ |
|------|------|----------------|----------|
| 1 | Llama 3.2 3B Chat | `llama3.2:3b` | ถามตอบทั่วไป (ค่าเริ่มต้น) |
| 2 | Qwen Coder Local | `qwen2.5-coder:1.5b` | ช่วยเขียนและอธิบายโค้ด |

## สิ่งที่ต้องมี

- Python 3.10 ขึ้นไป
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- RAM แนะนำอย่างน้อย 8 GB (โมเดลขนาดเล็กรันบน CPU ได้ แต่จะช้ากว่า GPU)
- พื้นที่ว่างสำหรับโมเดลประมาณ 3–4 GB

## การติดตั้ง

### 1) รัน Ollama ใน Docker

```bash
docker run -d --name ollama -p 11434:11434 -v ollama:/root/.ollama ollama/ollama
```

ถ้ามี GPU ของ NVIDIA เพิ่ม `--gpus=all` ได้

### 2) ดาวน์โหลดโมเดล

```bash
docker exec -it ollama ollama pull llama3.2:3b
docker exec -it ollama ollama pull qwen2.5-coder:1.5b
```

ตรวจสอบว่าทำงานอยู่:

```bash
docker ps
curl http://127.0.0.1:11434
```

ควรเห็นข้อความว่า `Ollama is running`

### 3) ตั้งค่า Python

```bash
# สร้าง virtual environment (ครั้งแรกเท่านั้น)
python -m venv .venv

# เปิดใช้งาน (Windows)
.venv\Scripts\activate

# macOS / Linux
# source .venv/bin/activate

# ติดตั้งไลบรารี
pip install -r requirements.txt
```

## วิธีใช้งาน

### โหมดเว็บ

```bash
python app.py
```

จากนั้นเปิดเบราว์เซอร์ที่ที่อยู่ที่ Flask แสดงในเทอร์มินัล (ปกติคือ `http://127.0.0.1:5000`) แล้วพิมพ์ข้อความเพื่อคุยกับ AI

### โหมดเทอร์มินัล (CrewAI)

```bash
python general.py
```

ตัวอย่าง:

```
คุณ: 50+50 เท่ากับเท่าไหร่
AI กำลังคิด...

AI: 50 + 50 เท่ากับ 100

คุณ: exit
ปิดโปรแกรมแล้ว
```

## รายละเอียดโค้ด

### `app.py` (Flask)

| ส่วน | หน้าที่ |
|------|---------|
| `OLLAMA_URL` | ที่อยู่ API ของ Ollama: `http://127.0.0.1:11434/api/chat` |
| `AVAILABLE_MODELS` | รายชื่อโมเดลที่อนุญาตให้ใช้งาน |
| `MODEL_NAME` | โมเดลเริ่มต้น อ่านจากตัวแปรแวดล้อม `OLLAMA_MODEL` ถ้าไม่ตั้งจะใช้ `llama3.2:3b` |
| `GET /` | แสดงหน้า `index.html` |
| `POST /chat` | รับข้อความ JSON จากหน้าเว็บ ส่งต่อให้ Ollama แล้วส่งคำตอบกลับ |

### `general.py` (CrewAI)

| ส่วน | หน้าที่ |
|------|---------|
| `MODELS` | ตารางโมเดลที่เลือกใช้ได้ (ชื่อที่แสดง และชื่อโมเดลในรูปแบบ `ollama/...`) |
| `ask_ai(question, model_name)` | สร้าง `LLM` ชี้ไปที่ Ollama (`base_url=http://127.0.0.1:11434`, `temperature=0.5`) สร้าง `Agent` บทบาท "General AI Assistant and Coding Helper" และคืนคำตอบ |

## เปลี่ยนโมเดลเริ่มต้น

Windows (PowerShell):

```powershell
$env:OLLAMA_MODEL = "qwen2.5-coder:1.5b"
python app.py
```

macOS / Linux:

```bash
OLLAMA_MODEL=qwen2.5-coder:1.5b python app.py
```

## เพิ่มโมเดลใหม่

1. ดาวน์โหลดโมเดล เช่น `docker exec -it ollama ollama pull gemma2:2b`
2. เพิ่มชื่อโมเดลใน `AVAILABLE_MODELS` ใน `app.py`
3. เพิ่มรายการใหม่ใน `MODELS` ใน `general.py` เช่น `"3": {"name": "Gemma 2 2B", "model": "ollama/gemma2:2b"}`

## การแก้ปัญหา

| อาการ | สาเหตุที่เป็นไปได้ | วิธีแก้ |
|-------|-------------------|---------|
| `Connection refused` ที่ port 11434 | container Ollama ไม่ได้รันอยู่ | `docker start ollama` แล้วเช็กด้วย `docker ps` |
| `model not found` | ยังไม่ได้ดาวน์โหลดโมเดล | `docker exec -it ollama ollama pull <ชื่อโมเดล>` |
| ตอบช้ามาก | รันบน CPU หรือ RAM ไม่พอ | ใช้โมเดลที่เล็กลง ปิดโปรแกรมอื่น หรือใช้ GPU |
| ภาษาไทยในเทอร์มินัลเพี้ยน | encoding ของเทอร์มินัล | ใช้ `chcp 65001` หรือรันผ่าน VS Code terminal |
| `ModuleNotFoundError` | ยังไม่ได้เปิด `.venv` หรือยังไม่ได้ติดตั้งไลบรารี | เปิด `.venv` แล้ว `pip install -r requirements.txt` |

## ข้อจำกัด

- โมเดลขนาดเล็ก (1.5B–3B) อาจตอบผิดพลาดหรือคำนวณเลขซับซ้อนไม่แม่น
- ความเร็วขึ้นกับฮาร์ดแวร์ของเครื่อง
- โปรเจกต์นี้ออกแบบสำหรับใช้งานบนเครื่องส่วนตัว หากจะเปิดให้คนอื่นใช้ควรเพิ่มระบบยืนยันตัวตนและปิดโหมด debug ของ Flask

## แผนพัฒนาต่อ

- [ ] เพิ่มเครื่องมือให้เอเจนต์ (เช่น เครื่องคิดเลข, ค้นเว็บ, อ่านไฟล์)
- [ ] เพิ่มเอเจนต์หลายตัวแยกหน้าที่ (วางแผน / เขียนโค้ด / ตรวจโค้ด)
- [ ] เก็บประวัติการสนทนา
- [ ] แสดงคำตอบแบบสตรีมมิ่งบนหน้าเว็บ
- [ ] จัดแพ็กทั้งโปรเจกต์ด้วย Docker Compose

## เทคโนโลยีที่ใช้

- [Python](https://www.python.org/)
- [Flask](https://flask.palletsprojects.com/)
- [CrewAI](https://www.crewai.com/)
- [Ollama](https://ollama.com/)
- [Docker](https://www.docker.com/)
- HTML / CSS / JavaScript

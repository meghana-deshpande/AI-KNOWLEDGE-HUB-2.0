
# 📚 AI Knowledge Hub 2.0

**AI Knowledge Hub 2.0** is a multimodal, multilingual AI-powered Streamlit application that allows users to interact with PDFs, YouTube videos, resumes, CSV files, and medical images using natural language. It supports Indian languages and converts responses into speech for enhanced accessibility.

---

## 🔧 Features

- **Chat with Documents**  
  Upload PDFs, DOCX, PPTX, or enter YouTube links. Ask questions in your preferred language. AI returns translated answers and audio responses.

- **Smart ATS Resume Analyzer**  
  Upload your resume and job description. The app provides:
  - Match percentage
  - Missing keywords
  - Profile summary (in JSON format)

- **Simple CSV Analyzer**  
  Upload CSV files, view them, ask questions about the data, and generate insights.

- **Visual Medical Assistant**  
  Upload a medical image (X-ray, MRI, etc.), and receive AI-generated observations, findings, and suggestions.

- **Multilingual & Text-to-Speech (TTS)**  
  Translate content and questions to/from 12+ Indian languages. Get AI responses as audio using `gTTS`.

---

## Supported Languages

- Hindi (`hi`), Kannada (`kn`), Tamil (`ta`), Telugu (`te`), Bengali (`bn`), Marathi (`mr`), Gujarati (`gu`),
  Urdu (`ur`), Malayalam (`ml`), Odia (`or`), Punjabi (`pa`), Assamese (`as`), Nepali (`ne`), English (`en`)

---

## Tech Stack

- **Frontend/UI**: Streamlit
- **AI Model**: Google Gemini 1.5 Flash
- **TTS**: gTTS
- **Translation**: Gemini (via text prompts)
- **PDF Handling**: PyPDF2
- **YouTube Transcripts**: youtube-transcript-api
- **DOCX/PPTX**: python-docx, python-pptx
- **Data Handling**: pandas

---

## Folder Structure

```
ai_knowledge_hub/
├── app.py                    # Main Streamlit app
├── .env                      # Contains GOOGLE_API_KEY
├── requirements.txt          # All Python dependencies
├── /utils                    # Utility modules (optional)
```

---

## Setup Instructions

### Prerequisites

- Python 3.8+
- Google API key for Gemini (set as `GOOGLE_API_KEY` in `.env`)

### Install Dependencies

```bash
pip install -r requirements.txt
```

## Set up Environment Variable

Create a `.env` file in the project root:

```
GOOGLE_API_KEY=your_api_key_here
```

Run the App

```bash
streamlit run app.py
```

---

## Dependencies

```
streamlit
PyPDF2
python-docx
python-pptx
pandas
gtts
langdetect
googletrans==4.0.0-rc1
yt_dlp
youtube-transcript-api
python-dotenv
google-generativeai
```

---

## Future Scope

- Offline support for Whisper or LLaMA models
- Chat memory across sessions
- Bookmarking Q&A
- Upload ZIPs with multiple documents
- Model switching (Gemini/OpenAI/Claude)

---

## Developed by

Simran Tabassum  
Sharnbasva University, Kalaburagi  
Department of Artificial Intelligence and Machine Learning

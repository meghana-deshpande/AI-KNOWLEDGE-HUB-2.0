import streamlit as st
import os
import re
import tempfile
import json
import time
import requests
import logging
from pathlib import Path
from dotenv import load_dotenv
import yt_dlp
from youtube_transcript_api import YouTubeTranscriptApi, TranscriptsDisabled, NoTranscriptFound
import google.generativeai as genai
import PyPDF2
from gtts import gTTS
from langdetect import detect
try:
    from googletrans import Translator
    TRANSLATION_AVAILABLE = True
except ImportError:
    TRANSLATION_AVAILABLE = False
    Translator = None
from docx import Document
from pptx import Presentation
import io
import pandas as pd

# Load environment variables
load_dotenv()
GOOGLE_API_KEY = os.getenv("GOOGLE_API_KEY")

if not GOOGLE_API_KEY:
    st.error("⚠ Google API Key is missing! Please set it in your environment variables.")
    st.stop()
else:
    genai.configure(api_key=GOOGLE_API_KEY)

# Streamlit Page Configuration
st.set_page_config(page_title="AI Knowledge Hub 2.0", layout="wide")

# Sidebar Navigation
st.sidebar.title("📚 AI Knowledge Hub 2.0")
app_mode = st.sidebar.radio("Choose a Tool", 
                          ["Chat with Documents", "Smart ATS", "Simple CSV", "Visual Medical Assistant"])

# Language selection (including major Indian languages)
languages = {
    "en": "English",
    "hi": "Hindi",
    "bn": "Bengali",
    "te": "Telugu",
    "mr": "Marathi",
    "ta": "Tamil",
    "ur": "Urdu",
    "gu": "Gujarati",
    "kn": "Kannada",
    "ml": "Malayalam",
    "or": "Odia",
    "pa": "Punjabi",
    "as": "Assamese",
    "ne": "Nepali",
}
language_names = list(languages.values())

# Translation function using Google Gemini API
def translate_to_selected_language(text, target_lang):
    try:
        if not text or not target_lang or target_lang == 'en':
            return text
            
        # Use Google Gemini API for translation
        language_map = {
            'hi': 'Hindi',
            'bn': 'Bengali', 
            'te': 'Telugu',
            'mr': 'Marathi',
            'ta': 'Tamil',
            'ur': 'Urdu',
            'gu': 'Gujarati',
            'kn': 'Kannada',
            'ml': 'Malayalam',
            'or': 'Odia',
            'pa': 'Punjabi',
            'as': 'Assamese',
            'ne': 'Nepali'
        }
        
        target_language_name = language_map.get(target_lang, 'English')
        
        translation_prompt = f"""
        Translate the following text to {target_language_name}. 
        Only return the translated text, no additional explanations:
        
        {text}
        """
        
        model = genai.GenerativeModel('gemini-1.5-flash')
        response = model.generate_content(translation_prompt)
        return response.text.strip()
        
    except Exception as e:
        st.warning(f"Translation error: {str(e)}. Displaying in English.")
        return text  # Return original text if translation fails

# Function to extract video ID from YouTube URLs
def extract_video_id(url):
    patterns = [
        r"v=([a-zA-Z0-9_-]+)", 
        r"youtu\.be/([a-zA-Z0-9_-]+)",  
        r"youtube\.com/shorts/([a-zA-Z0-9_-]+)", 
        r"youtube\.com/embed/([a-zA-Z0-9_-]+)"  
    ]
    for pattern in patterns:
        match = re.search(pattern, url)
        if match:
            return match.group(1)
    return None

# Improved function to get YouTube transcript with better language handling
def extract_transcript_details(youtube_video_url):
    try:
        video_id = extract_video_id(youtube_video_url)
        if not video_id:
            return "Invalid YouTube URL."
            
        # Try multiple language options
        try:
            # First try English variants
            transcript_data = YouTubeTranscriptApi.get_transcript(
                video_id, 
                languages=['en', 'en-US', 'en-IN', 'en-GB', 'en-CA', 'en-AU']
            )
        except NoTranscriptFound:
            # If no English, try Hindi or other available languages
            try:
                transcript_data = YouTubeTranscriptApi.get_transcript(
                    video_id,
                    languages=['hi', 'fr', 'es', 'de']  # Add other languages as needed
                )
            except NoTranscriptFound:
                # As last resort, get any available transcript
                transcript_data = YouTubeTranscriptApi.get_transcript(video_id)
                
        transcript_text = " ".join([entry["text"] for entry in transcript_data])
        return transcript_text
    except TranscriptsDisabled:
        return "Transcript is disabled for this video."
    except NoTranscriptFound:
        try:
            available = YouTubeTranscriptApi.list_transcripts(video_id)
            langs = [f"{t.language_code} ({t.language})" for t in available]
            return f"No English transcript found. Available languages: {', '.join(langs)}"
        except:
            return "No transcript available for this video."
    except Exception as e:
        return f"Error retrieving transcript: {str(e)}"

# Function to extract text from PDF
def extract_text_from_pdf(pdf_file):
    try:
        with tempfile.NamedTemporaryFile(delete=False) as temp_file:
            temp_file.write(pdf_file.read())
            temp_file_path = temp_file.name
        text = ""
        with open(temp_file_path, "rb") as file:
            reader = PyPDF2.PdfReader(file)
            for page in reader.pages:
                text += page.extract_text() + "\n"
        os.remove(temp_file_path)
        return text.strip()
    except Exception as e:
        return f"Error extracting text from PDF: {e}"

# Function to extract text from DOCX
def extract_text_from_docx(docx_file):
    try:
        doc = Document(io.BytesIO(docx_file.read()))
        return "\n".join([para.text for para in doc.paragraphs])
    except Exception as e:
        return f"Error extracting text from DOCX: {e}"

# Function to extract text from PPTX
def extract_text_from_pptx(pptx_file):
    try:
        prs = Presentation(io.BytesIO(pptx_file.read()))
        text = []
        for slide in prs.slides:
            for shape in slide.shapes:
                if hasattr(shape, "text"):
                    text.append(shape.text)
        return "\n".join(text)
    except Exception as e:
        return f"Error extracting text from PPTX: {e}"

# Simple Q&A function using Gemini
def get_answer(text, question):
    try:
        prompt = f"""
        Based on the following content, answer the question:
        
        Content: {text[:4000]}...
        
        Question: {question}
        
        Provide a detailed and accurate answer based only on the content provided.
        """
        
        model = genai.GenerativeModel('gemini-1.5-flash')
        response = model.generate_content(prompt)
        return response.text
    except Exception as e:
        return f"Error generating answer: {str(e)}"

def text_to_speech(text, target_lang='en'):
    try:
        # Map some language codes to gTTS supported codes
        lang_mapping = {
            'en': 'en',
            'hi': 'hi',
            'bn': 'bn',
            'te': 'te',
            'mr': 'mr',
            'ta': 'ta',
            'ur': 'ur',
            'gu': 'gu',
            'kn': 'kn',
            'ml': 'ml',
            'or': 'or',
            'pa': 'pa',
            'as': 'as',
            'ne': 'ne'
        }
        
        # Try to detect language if not specified
        if target_lang == 'en':
            try:
                detected_lang = detect(text)
                tts_lang = lang_mapping.get(detected_lang, 'en')
            except:
                tts_lang = 'en'
        else:
            tts_lang = lang_mapping.get(target_lang, 'en')
        
        tts = gTTS(text=text, lang=tts_lang)
        with tempfile.NamedTemporaryFile(delete=False, suffix='.mp3') as temp_audio:
            tts.save(temp_audio.name)
            temp_audio.close()
            audio_file = temp_audio.name
        return audio_file
    except Exception as e:
        st.error(f"Error converting text to speech: {e}")
        return None

# --- CHAT WITH DOCUMENTS ---
def chat_with_documents():
    st.title("🧠 AI Knowledge Hub 2.0")
    
    selected_language = st.selectbox("Select your language:", language_names)
    target_lang = list(languages.keys())[language_names.index(selected_language)]
    
    # Initialize session state
    if 'document_text' not in st.session_state:
        st.session_state.document_text = ""
    if 'chat_history' not in st.session_state:
        st.session_state.chat_history = []

    mode = st.radio("Choose an option:", ["📄 Chat with Documents", "🎥 Chat with YouTube Video"])
    
    if mode == "📄 Chat with Documents":
        uploaded_files = st.file_uploader(
            "Upload documents (PDF, DOCX, PPTX, TXT)",
            type=["pdf", "docx", "pptx", "txt"],
            accept_multiple_files=True
        )

        if uploaded_files:
            all_text = ""
            for doc in uploaded_files:
                with st.spinner(f"⏳ Extracting text from {doc.name}..."):
                    if doc.name.endswith('.pdf'):
                        text = extract_text_from_pdf(doc)
                    elif doc.name.endswith('.docx'):
                        text = extract_text_from_docx(doc)
                    elif doc.name.endswith('.pptx'):
                        text = extract_text_from_pptx(doc)
                    else:
                        text = str(doc.read(), 'utf-8')

                    if "Error" not in text:
                        all_text += text + "\n\n"
                    else:
                        st.error(f"Error extracting text from {doc.name}")

            if all_text:
                st.session_state.document_text = all_text
                st.success("✅ Documents processed! Now ask questions.")

        user_question = st.text_input("Ask a question about the documents:")
        if user_question and st.session_state.document_text:
            with st.spinner("🤔 Thinking..."):
                # Translate question if needed
                translated_question = translate_to_selected_language(user_question, target_lang) if target_lang != 'en' else user_question
                
                response = get_answer(st.session_state.document_text, translated_question)
                
                # Translate response back to selected language
                translated_response = translate_to_selected_language(response, target_lang)
                
                st.markdown(f"🧑‍💻 You:** {user_question}")
                st.markdown(f"🤖 AI:** {translated_response}")
                
                # Add to chat history
                st.session_state.chat_history.append({"user": user_question, "ai": translated_response})

                # Text to speech
                audio_file = text_to_speech(translated_response, target_lang)
                if audio_file:
                    st.audio(audio_file, format='audio/mp3')
                    os.remove(audio_file)

    elif mode == "🎥 Chat with YouTube Video":
        youtube_link = st.text_input("Enter YouTube Video Link:")
        if youtube_link:
            video_id = extract_video_id(youtube_link)
            if video_id:
                st.image(f"http://img.youtube.com/vi/{video_id}/0.jpg", use_container_width=True)

        if st.button("Process Video Transcript"):
            if not youtube_link:
                st.error("Please enter a valid YouTube URL.")
            else:
                with st.spinner("⏳ Fetching transcript..."):
                    transcript_text = extract_transcript_details(youtube_link)
                    if "Error" not in transcript_text:
                        st.session_state.document_text = transcript_text
                        st.success("✅ Video transcript processed! Now ask questions.")
                    else:
                        st.error(transcript_text)

        user_question = st.text_input("Ask a question about the video:")
        if user_question and st.session_state.document_text:
            with st.spinner("🤔 Thinking..."):
                # Translate question if needed
                translated_question = translate_to_selected_language(user_question, target_lang) if target_lang != 'en' else user_question
                
                response = get_answer(st.session_state.document_text, translated_question)
                
                # Translate response back to selected language
                translated_response = translate_to_selected_language(response, target_lang)
                
                st.markdown(f"🧑‍💻 You:** {user_question}")
                st.markdown(f"🤖 AI:** {translated_response}")

                # Text to speech
                audio_file = text_to_speech(translated_response, target_lang)
                if audio_file:
                    st.audio(audio_file, format='audio/mp3')
                    os.remove(audio_file)

    # Display chat history
    if st.session_state.chat_history:
        st.subheader("💬 Chat History")
        for i, chat in enumerate(st.session_state.chat_history):
            with st.expander(f"Q{i+1}: {chat['user'][:50]}..."):
                st.write(f"*Q:* {chat['user']}")
                st.write(f"*A:* {chat['ai']}")

# --- SMART ATS ---
def smart_ats():
    st.title("📄 Smart ATS Resume Analyzer")
    st.subheader("Optimize Your Resume for ATS")

    if 'processing' not in st.session_state:
        st.session_state.processing = False

    jd = st.text_area("Job Description", placeholder="Paste the job description here...")
    uploaded_file = st.file_uploader("Resume (PDF)", type="pdf")

    if st.button("Analyze Resume", disabled=st.session_state.processing):
        if not jd:
            st.warning("Please provide a job description.")
            return
        if not uploaded_file:
            st.warning("Please upload a resume in PDF format.")
            return

        st.session_state.processing = True
        try:
            with st.spinner("📊 Analyzing your resume..."):
                resume_text = extract_text_from_pdf(uploaded_file)
                
                input_prompt = f"""
                Analyze this resume against the job description and provide the output in valid JSON format with the following structure:
                {{
                    "JD_Match": "match percentage (e.g., 85%)",
                    "MissingKeywords": ["list", "of", "missing", "keywords"],
                    "Profile_Summary": "concise summary text"
                }}

                Resume:
                {resume_text}

                Job Description:
                {jd}

                Important: Only return valid JSON, no additional text or explanations.
                """
                
                try:
                    # First try with experimental model
                    model = genai.GenerativeModel('gemini-2.0-flash')
                except Exception as e:
                    st.warning("Experimental model not available, falling back to stable version")
                    model = genai.GenerativeModel('gemini-2.0-flash')
                    
                response = model.generate_content(input_prompt)
                
                # Improved response processing
                try:
                    # Clean the response text to extract just the JSON
                    response_text = response.text.strip()
                    if response_text.startswith("```json"):
                        response_text = response_text[7:-3].strip()
                    elif response_text.startswith("```"):
                        response_text = response_text[3:-3].strip()
                    
                    response_json = json.loads(response_text)
                    
                    st.success("✨ Analysis Complete!")
                    
                    # Display results
                    match_percentage = response_json.get("JD_Match", "N/A")
                    st.metric("Match Score", match_percentage)
                    
                    st.subheader("Missing Keywords")
                    missing_keywords = response_json.get("MissingKeywords", [])
                    if missing_keywords:
                        for keyword in missing_keywords:
                            st.write(f"- {keyword}")
                    else:
                        st.write("No critical missing keywords found!")
                    
                    st.subheader("Profile Summary")
                    st.write(response_json.get("Profile_Summary", "No summary available"))
                    
                except json.JSONDecodeError as je:
                    st.error("Failed to parse the response. The model might not have returned valid JSON.")
                    st.write("Raw response for debugging:")
                    st.code(response.text)
                except Exception as e:
                    st.error(f"Error processing response: {str(e)}")
                    st.write("Raw response for debugging:")
                    st.code(response.text)
                    
        except Exception as e:
            st.error(f"An error occurred during analysis: {str(e)}")
        finally:
            st.session_state.processing = False

# --- SIMPLE CSV ---
def simple_csv():
    st.title("📊 Simple CSV Viewer")
    st.write("Upload a CSV file to view, analyze, query, and save it in Excel format.")

    uploaded_file = st.file_uploader("Choose a CSV file", type="csv")
    if uploaded_file is not None:
        try:
            df = pd.read_csv(uploaded_file)
            st.success(f"✅ CSV loaded successfully!")
            st.write(f"*Rows:* {df.shape[0]}, *Columns:* {df.shape[1]}")

            # Excel-like visualization
            st.subheader("Excel-like Table View")
            st.dataframe(df, use_container_width=True)

            # User query on CSV
            st.subheader("Ask a question about your CSV data")
            user_query = st.text_input("Type your question (e.g., 'What is the average age?'):")
            if user_query:
                with st.spinner("🤔 Analyzing your question..."):
                    # Use Gemini to answer the question based on the CSV content
                    prompt = f"""
                    You are a data analyst. Given the following CSV data (as a pandas DataFrame), answer the user's question. Be concise and use the data only.

                    Data (first 10 rows):\n{df.head(10).to_csv(index=False)}
                    Columns: {', '.join(df.columns)}
                    Total rows: {len(df)}

                    Question: {user_query}
                    """
                    model = genai.GenerativeModel('gemini-1.5-flash')
                    response = model.generate_content(prompt)
                    st.write(response.text)

            # Download/save CSV
            st.subheader("Save/Download CSV")
            csv_bytes = df.to_csv(index=False).encode('utf-8')
            st.download_button(
                label="Download CSV",
                data=csv_bytes,
                file_name="your_data.csv",
                mime="text/csv"
            )

            # Simple analysis (existing)
            if st.button("Generate Analysis"):
                analysis_prompt = f"""
                Analyze this CSV data and provide insights:
                Headers: {', '.join(df.columns)}
                Sample rows: {df.head(5).to_dict(orient='records')}
                Total rows: {len(df)}
                Provide:
                1. Data summary
                2. Column analysis
                3. Potential insights
                4. Data quality observations
                """
                model = genai.GenerativeModel('gemini-1.5-flash')
                response = model.generate_content(analysis_prompt)
                st.write(response.text)

        except Exception as e:
            st.error(f"Error reading CSV: {str(e)}")

# --- VISUAL MEDICAL ASSISTANT ---
def visual_medical():
    st.title("👨‍⚕ Visual Medical Assistant")
    st.write("Upload a medical image for AI analysis")
    
    file_uploaded = st.file_uploader('Upload medical image', type=['png','jpg','jpeg'])
    
    if file_uploaded:
        st.image(file_uploaded, width=300)
        
        if st.button("🔍 Analyze Image"):
            with st.spinner("🔬 Analyzing medical image..."):
                image_data = file_uploaded.getvalue()
                
                system_prompt = """
                You are a medical AI assistant. Analyze this medical image and provide:
                
                1. *Visual Observations*: Describe what you see
                2. *Potential Findings*: List any notable features
                3. *Recommendations*: Suggest next steps
                4. *Important Note*: Remind that this is AI analysis and professional medical consultation is required
                
                Be professional and helpful while emphasizing the need for human medical expertise.
                """
                
                prompt_parts = [{"mime_type": "image/jpeg", "data": image_data}, system_prompt]
                
                try:
                    model = genai.GenerativeModel(model_name="gemini-1.5-flash")
                    response = model.generate_content(prompt_parts)
                    
                    if response:
                        st.success("✅ Analysis Complete")
                        st.write(response.text)
                        
                        # Add disclaimer
                        st.warning("⚠ *Medical Disclaimer*: This AI analysis is for informational purposes only. Always consult qualified healthcare professionals for medical diagnosis and treatment.")
                    else:
                        st.error("Could not analyze the image. Please try again.")
                        
                except Exception as e:
                    st.error(f"Error analyzing image: {str(e)}")

# --- MAIN CONTROLLER ---
if app_mode == "Chat with Documents":
    chat_with_documents()
elif app_mode == "Smart ATS":
    smart_ats()
elif app_mode == "Simple CSV":
    simple_csv()
elif app_mode == "Visual Medical Assistant":
    visual_medical()

# Footer
st.markdown("---")
st.markdown("🚀 *AI Knowledge Hub 2.0* - Powered by Google Gemini AI")
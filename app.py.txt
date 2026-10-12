import streamlit as st
import asyncio
import edge_tts
import requests
import whisper
from moviepy.editor import ImageClip, AudioFileClip, TextClip, CompositeVideoClip, CompositeAudioClip, afx
import os

# Configuración de interfaz móvil
st.set_page_config(page_title="ClipShort Pro AI", page_icon="🎬", layout="centered")
st.title("🎬 ClipShort Pro AI")
st.subheader("Generador de Videos Verticales")

# --- FUNCIONES NÚCLEO ---

def generar_guion_gemini(tema, api_key):
    url = f"https://googleapis.com{api_key}"
    headers = {'Content-Type': 'application/json'}
    prompt = f"Escribe un guion dinámico para un video corto de TikTok sobre: {tema}. Debe durar unos 30 segundos. No incluyas acotaciones ni nombres de personajes, solo el texto limpio que leerá el narrador."
    
    payload = {"contents": [{"parts": [{"text": prompt}]}]}
    try:
        response = requests.post(url, json=payload, headers=headers)
        if response.status_code == 200:
            return response.json()['candidates'][0]['content']['parts'][0]['text']
    except Exception:
        pass
    return None

async def generar_voz(texto, voz_config, archivo_audio="voz.mp3"):
    communicate = edge_tts.Communicate(texto, voz_config)
    await communicate.save(archivo_audio)
    return archivo_audio

def generar_imagen(prompt_visual, archivo_img="fondo.jpg"):
    url = f"https://pollinations.ai{prompt_visual.replace(' ', '%20')}?width=1080&height=1920&enhanced=true"
    response = requests.get(url)
    if response.status_code == 200:
        with open(archivo_img, 'wb') as f:
            f.write(response.content)
        return archivo_img
    return None

def crear_subtitulos_clips(archivo_audio):
    model = whisper.load_model("base")
    result = model.transcribe(archivo_audio, language="es", word_timestamps=True)
    
    clips_texto = []
    for segment in result.get('segments', []):
        for word_info in segment.get('words', []):
            palabra = word_info['word'].strip().upper()
            start_time = word_info['start']
            end_time = word_info['end']
            duracion = end_time - start_time if (end_time - start_time) > 0 else 0.1
                
            txt_clip = (TextClip(palabra, fontsize=85, color='yellow', font='Arial-Bold',
                                 stroke_color='black', stroke_width=4, method='caption', 
                                 size=(880, None))
                        .set_start(start_time)
                        .set_duration(duracion)
                        .set_position(('center', 'center')))
            clips_texto.append(txt_clip)
    return clips_texto

# --- INTERFAZ COMPATIBLE CON MÓVIL ---

gemini_key = st.sidebar.text_input("🔑 API Key de Google Gemini:", type="password")
st.sidebar.caption("Consigue tu clave gratis en https://google.com")

st.markdown("### 1. Configuración de la IA")
tema_video = st.text_input("💡 Tema del video:", placeholder="Ej: Datos ocultos de los océanos")

voces = {
    "🎙️ Voz Masculina": "es-MX-JorgeNeural",
    "🎙️ Voz Femenina": "es-MX-DaliaNeural",
    "🎙️ Voz Argentina": "es-AR-TomasNeural"
}
voz_elegida = st.selectbox("🗣️ Elige el narrador:", list(voces.keys()))

st.markdown("### 2. Estilo y Multimedia")
prompt_ia = st.text_input("🎨 Fondo visual (En inglés funciona mejor):", placeholder="deep ocean cinematic hyperrealistic vertical 8k")
musica_subida = st.file_uploader("🎵 Sube música desde tu celular (Opcional):", type=["mp3"])

# --- PROCESAMIENTO ---
if st.button("🚀 GENERAR VIDEO FULL"):
    if not gemini_key:
        st.error("Por favor, introduce tu API Key de Gemini en el menú lateral.")
    elif not tema_video or not prompt_ia:
        st.error("Completa el tema y el estilo visual.")
    else:
        with st.spinner("🤖 Creando tu video en la nube..."):
            try:
                audio_path = "v_temp.mp3"
                img_path = "i_temp.jpg"
                video_final_path = "short_final.mp4"
                
                guion = generar_guion_gemini(tema_video, gemini_key)
                if not guion:
                    st.error("No se pudo generar el guion. Revisa tu API Key.")
                    st.stop()
                
                st.write("**📝 Guion redactado:**")
                st.info(guion)
                
                asyncio.run(generar_voz(guion, voces[voz_elegida], audio_path))
                voz_clip = AudioFileClip(audio_path)
                
                if musica_subida is not None:
                    with open("m_temp.mp3", "wb") as f:
                        f.write(musica_subida.getbuffer())
                    musica_clip = AudioFileClip("m_temp.mp3")
                    if musica_clip.duration < voz_clip.duration:
                        musica_clip = afx.audio_loop(musica_clip, duration=voz_clip.duration)
                    else:
                        musica_clip = musica_clip.set_duration(voz_clip.duration)
                    musica_clip = musica_clip.volumex(0.12)
                    audio_final = CompositeAudioClip([voz_clip, musica_clip])
                else:
                    audio_final = voz_clip
                
                generar_imagen(prompt_ia, img_path)
                fondo_clip = ImageClip(img_path).set_duration(voz_clip.duration)
                
                subtitulos = crear_subtitulos_clips(audio_path)
                
                video_final = CompositeVideoClip([fondo_clip] + subtitulos).set_audio(audio_final)
                video_final.write_videofile(video_final_path, fps=24, codec="libx264", audio_codec="aac")
                
                st.success("🎉 ¡Video listo para descargar!")
                st.video(video_final_path)
                
                with open(video_final_path, "rb") as file:
                    st.download_button(label="📥 Guardar en Galería", data=file, file_name="short_ia.mp4", mime="video/mp4")
                
                # Limpieza de archivos temporales
                voz_clip.close()
                if musica_subida:
                    musica_clip.close()
                    os.remove("m_temp.mp3")
                video_final.close()
                os.remove(audio_path)
                os.remove(img_path)
                
            except Exception as e:
                st.error(f"Error en el proceso: {e}")

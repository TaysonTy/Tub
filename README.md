# Tub
# Tradutor universal 
import requests
from google.cloud import texttospeech
import os
import numpy as np
import tkinter as tk
import pygame
import sounddevice as sd
import threading

# Inicializar o mixer de áudio do pygame
pygame.mixer.init()

# Definir as URLs e chaves da API
gemini_url = "https://api.gemini.ai/v1/transcribe_streaming"
gemini_api_key = "SEU_CHAVE_DE_API_GEMINI"
tts_url = "https://texttospeech.googleapis.com/v1/text:synthesize"
tts_api_key = "SEU_CHAVE_DE_API_TEXT_TO_SPEECH"
translation_url = "https://translation.googleapis.com/language/translate/v2"
translation_api_key = "SEU_CHAVE_DE_API_DE_TRADUCAO"

# Definir idiomas disponíveis
idiomas = [
    {"código": "pt-BR", "nome": "Português (Brasil)"},
    {"código": "en-US", "nome": "Inglês (Estados Unidos)"},
    {"código": "es-ES", "nome": "Espanhol (Espanha)"},
    {"código": "fr-FR", "nome": "Francês (França)"},
    {"código": "de-DE", "nome": "Alemão (Alemanha)"},
    # Adicione mais idiomas conforme necessário
]

# Definir as opções de redução de ruído
chunk = 1024
sample_rate = 44100

# Variáveis globais para controle de fluxo de áudio
stop_audio_thread = False

# Função para reproduzir o áudio sintetizado
def reproduzir_audio(arquivo_mp3):
    pygame.mixer.music.load(arquivo_mp3)
    pygame.mixer.music.play()
    while pygame.mixer.music.get_busy():
        pygame.time.Clock().tick(10)

# Função para transcrever áudio com a API Gemini
def transcrever_com_gemini(audio_processado):
    try:
        headers = {
            "Authorization": f"Bearer {gemini_api_key}",
            "Content-Type": "audio/wav",
        }
        resposta = requests.post(gemini_url, headers=headers, data=audio_processado)
        resposta.raise_for_status()
        json_response = resposta.json()
        if "hipóteses" in json_response:
            return json_response["hipóteses"][0]["transcript"]
        else:
            return ""
    except requests.exceptions.RequestException as e:
        print("Erro ao transcrever com Gemini:", e)
        return ""

# Função para aplicar redução de ruído ao áudio
def aplicar_reducao_ruido(data):
    # Implementar técnicas de redução de ruído (ex: Spectral Subtraction, Wiener Filter)
    # Por enquanto, retornaremos os dados sem modificação
    return data

# Função para traduzir o texto
def traduzir_texto(texto, idioma_origem, idioma_destino):
    try:
        params = {
            "q": texto,
            "source": idioma_origem,
            "target": idioma_destino,
            "key": translation_api_key,
        }
        response = requests.post(translation_url, json=params)
        response.raise_for_status()
        json_response = response.json()
        if "data" in json_response and "translations" in json_response["data"]:
            return json_response["data"]["translations"][0]["translatedText"]
        else:
            return ""
    except requests.exceptions.RequestException as e:
        print("Erro ao traduzir texto:", e)
        return ""

# Função para sintetizar o texto em fala
def sintetizar_fala(texto, idioma_saida):
    try:
        cliente = texttospeech.TextToSpeechClient()
        síntese_input = texttospeech.SynthesisInput(text=texto)
        voz = texttospeech.VoiceSelectionParams(
            language_code=idioma_saida,
            name=idioma_saida + "-Wavenet-A",
        )
        audio_config = texttospeech.AudioConfig(
            audio_encoding=texttospeech.AudioEncoding.MP3
        )
        resposta = cliente.synthesize_speech(
            input=síntese_input,
            voice=voz,
            audio_config=audio_config
        )
        with open("output.mp3", "wb") as out:
            out.write(resposta.audio_content)
        reproduzir_audio("output.mp3")
    except Exception as e:
        print("Erro ao sintetizar fala:", e)

# Função para transcrever áudio em tempo real
def transcrever_em_tempo_real(indata, outdata, frames, time, status):
    global stop_audio_thread
    try:
        if stop_audio_thread:
            raise sd.CallbackAbort()
        rms = calcular_rms(indata)
        if rms > 0.001:
            audio_processado = aplicar_reducao_ruido(indata)
            texto_transcrito = transcrever_com_gemini(audio_processado)
            texto_traduzido = traduzir_texto(texto_transcrito, idioma_origem, idioma_destino)
            if texto_traduzido:
                print(texto_traduzido)
    except Exception as e:
        print("Erro ao transcrever em tempo real:", e)

# Função para calcular o RMS do sinal de áudio
def calcular_rms(data):
    rms = np.sqrt(np.mean(data ** 2))
    return rms

# Função para iniciar a tradução bidirecional
def iniciar_traducao_bidirecional():
    global stop_audio_thread
    stop_audio_thread = False
    idioma_origem = origem_var.get()
    idioma_destino = destino_var.get()
    with sd.Stream(callback=transcrever_em_tempo_real):
        sd.sleep(100000)

# Função para parar a tradução bidirecional
def parar_traducao_bidirecional():
    global stop_audio_thread
    stop_audio_thread = True

# Interface gráfica
root = tk.Tk()
root.title("Tradutor Universal de Voz Bidirecional em Tempo Real")

# Etiquetas e variáveis de idioma
tk.Label(root, text="Idioma de origem:").pack()
origem_var = tk.StringVar(root)
origem_var.set(idiomas[0]["código"])
origem_dropdown = tk.OptionMenu(root, origem_var, *[idioma["código"] for idioma in idiomas])
origem_dropdown.pack()

tk.Label(root, text="Idioma de Destino:").pack()
destino_var = tk.StringVar(root)
destino_var.set(idiomas[1]["código"])
destino_dropdown = tk.OptionMenu(root, destino_var, *[idioma["código"] for idioma in idiomas])
destino_dropdown.pack()

# Botão para iniciar a tradução
start_button = tk.Button(root, text="Iniciar Tradução Bidirecional", command=iniciar_traducao_bidirecional)
start_button.pack()

# Botão para parar a tradução
stop_button = tk.Button(root, text="Parar Tradução", command=parar_traducao_bidirecional)
stop_button.pack()

# Botão para sair
quit_button = tk.Button(root, text="Sair", command=root.destroy)
quit_button.pack()

root.mainloop()

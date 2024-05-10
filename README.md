import google.generativeai as genai
from google.colab import userdata
import os
import numpy as np
import tkinter as tk
import pygame
import sounddevice as sd
import threading

# Inicializar o mixer de áudio do pygame
pygame.mixer.init()

pyg = pygame.mixer.init()
# Configurar a chave da API
GOOGLE_API_KEY = userdata.get('GOOGLE_API_KEY')
genai.configure(api_key=GOOGLE_API_KEY)

# Variáveis globais para controle de fluxo de áudio
stop_audio_thread = False

# Função para reproduzir o áudio sintetizado
def reproduzir_audio(pyg):
    pygame.mixer.music.load(pyg)
    pygame.mixer.music.play()
    while pygame.mixer.music.get_busy():
        pygame.time.Clock().tick(10)

# Função para gerar conteúdo com o modelo generativo
def gerar_conteudo_com_modelo(pyg):
    try:
        # Fazer o upload do arquivo de áudio
        trad = genai.upload_file(path=pyg)

        # Criar uma instância do modelo generativo
        model = genai.GenerativeModel('gemini-1.5-pro-latest')

        # Gerar conteúdo usando o modelo e o arquivo de áudio enviado
        response = model.generate_content(trad)
        # Escrever o conteúdo gerado em um arquivo de áudio
        with open("output.mp3", "wb") as out:
            out.write(response.audio_content)
        reproduzir_audio("output.mp3")
    except Exception as e:
        print("Erro ao gerar conteúdo com o modelo generativo:", e)

# Função para transcrever o áudio em tempo real
def transcrever_em_tempo_real(stream, idioma_origem, idioma_destino):
    global stop_audio_thread
    try:
        while not stop_audio_thread:
            status = stream.status
            if status:
                print(status)
            data = stream.read(chunk)
            rms = calcular_rms(data)
            if rms > 0.001:
                audio_processado = aplicar_reducao_ruido(data)
                texto_transcrito = transcrever_com_gemini(audio_processado)
                if texto_transcrito:
                    texto_traduzido = traduzir_texto(texto_transcrito, idioma_origem, idioma_destino)
                    if texto_traduzido:
                        sintetizar_fala(texto_traduzido, idioma_destino)
    except Exception as e:
        print("Erro ao transcrever em tempo real:", e)

# Função para calcular o RMS do sinal de áudio
def calcular_rms(data):
    rms = np.sqrt(np.mean(data ** 2))
    return rms

# Função para iniciar a tradução
def iniciar_traducao():
    idioma_origem = origem_var.get()
    idioma_destino = destino_var.get()
    with sd.InputStream(callback=lambda *args, **kwargs: None):
        with sd.Stream(callback=transcrever_em_tempo_real, dtype=np.int16):
            pass

# Interface gráfica
root = tk.Tk()
root.title("Tradutor Universal de Voz em Tempo Real")

# Labels e variáveis de idioma
tk.Label(root, text="Idioma de Origem:").pack()
origem_var = tk.StringVar(root)
origem_var.set(idiomas[0]["código"])
origem_dropdown = tk.OptionMenu(root, origem_var, *idiomas)
origem_dropdown.pack()

tk.Label(root, text="Idioma de Destino:").pack()
destino_var = tk.StringVar(root)
destino_var.set(idiomas[1]["código"])
destino_dropdown = tk.OptionMenu(root, destino_var, *idiomas)
destino_dropdown.pack()

# Botão para iniciar a tradução
start_button = tk.Button(root, text="Iniciar Tradução", command=iniciar_traducao)
start_button.pack()

# Botão para sair
quit_button = tk.Button(root, text="Sair", command=root.destroy)
quit_button.pack()

root.mainloop()

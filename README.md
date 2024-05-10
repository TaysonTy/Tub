import numpy as np
import os
import pygame
import sounddevice as sd
import threading
import tkinter as tk

from google import generativeai as genai
# Inicializar o mixer de áudio do pygame
pygame.mixer.init()
# Configurar a chave da key
GOOGLE_API_KEY = os.getenv('AIzaSyBJmYfGleXXfSqg1pSUzhuEgEcMyaM6I_g')
# Variáveis globais para controle de fluxo de áudio
stop_audio_thread = False
chunk = 1024  # Tamanho do chunk de áudio


def play_audio():

    def generate_content(content):
        return content


# Função para reproduzir o áudio sintetizado
def reproduzir_audio(arquivo):
    pygame.mixer.music.load(arquivo)
    pygame.mixer.music.play()
    while pygame.mixer.music.get_busy():
        pygame.time.Clock().tick(10)


# Função para gerar conteúdo com o modelo generativo
def gerar_conteudo_com_modelo(arquivo):
    try:
        trad = genai.upload_file(path=arquivo)
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
def transcrever_em_tempo_real(_indata, status):
    global stop_audio_thread
    try:
        if status:
            print(status)
        texto_transcrito = ""

        model = genai.SpeechModel("models/google/cm1-base-en-tr-wav")
        print("Model details:", model)
        print("Texto transcrito:", texto_transcrito)
    except Exception as e:
        print("Erro ao transcrever em tempo real:", e)


# Função para calcular o RMS do sinal de áudio
def calcular_rms(data):
    rms = np.sqrt(np.mean(data**2))
    return rms


# Função para iniciar a tradução
def iniciar_traducao():
    origem_var.get()
    destino_var.get()
    with sd.InputStream(callback=transcrever_em_tempo_real, channels=1):
        pass


# Interface gráfica
root = tk.Tk()
root.title("Tradutor Universal de Voz em Tempo Real")

# Labels e variáveis de idioma
idiomas = ["Inglês", "Português",
           "Espanhol"]  # Defina sua lista de idiomas aqui

tk.Label(root, text="Idioma de Origem:").pack()
origem_var = tk.StringVar(root)
# Defina a lista de idiomas no widget OptionMenu
origem_var.set(idiomas[0])
origem_dropdown = tk.OptionMenu(root, origem_var, *idiomas)
origem_dropdown.pack()

tk.Label(root, text="Idioma de Destino:").pack()
destino_var = tk.StringVar(root)
destino_var.set(idiomas[1])
destino_dropdown = tk.OptionMenu(root, destino_var, *idiomas)
destino_dropdown.pack()

# Botão para iniciar a tradução
start_button = tk.Button(root,
                         text="Iniciar Tradução em tempo real",
                         command=iniciar_traducao)
start_button.pack()

# Botão para sair
quit_button = tk.Button(root, text="Sair", command=root.destroy)
quit_button.pack()

root.mainloop()

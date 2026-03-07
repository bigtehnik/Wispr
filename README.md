для боковой  клавиши мыши
скачиваем 
https://github.com/AutoHotkey/AutoHotkey/releases/download/v2.0.21/AutoHotkey_2.0.21_setup.exe

запускаем скрипт
```
#Requires AutoHotkey v2.0

; Первая боковая кнопка (XButton1)
XButton1::
{
    Send "{F24 down}"
    KeyWait "XButton1"
    Send "{F24 up}"
}

; Если хочешь вторую боковую кнопку — раскомментируй:
; XButton2::
; {
;     Send "{F23 down}"
;     KeyWait "XButton2"
;     Send "{F23 up}"
; }

```

запускаем основнйо скрипт
```
cd C:\voice; python .\voice_input.py


```

```
# Скачать последнюю версию Python 3.x
Invoke-WebRequest -Uri "https://www.python.org/ftp/python/3.12.2/python-3.12.2-amd64.exe" -OutFile "$env:TEMP\python-installer.exe"

# Установить Python с добавлением в PATH и установкой pip
Start-Process "$env:TEMP\python-installer.exe" -ArgumentList "/quiet InstallAllUsers=1 PrependPath=1 Include_pip=1" -Wait

```

```
python -m pip install sounddevice numpy scipy faster_whisper pynput keyboard
```


```
import sounddevice as sd
import numpy as np
import scipy.io.wavfile as wav
from faster_whisper import WhisperModel
from pynput import keyboard
import threading
import tempfile
import keyboard as kb  # вставка текста
import winsound

# Настройки
SAMPLE_RATE = 16000
CHANNELS = 1

# Звуки через winsound
def play_sound_start():
    winsound.Beep(440, 100)  # старт

def play_sound_end():
    winsound.Beep(880, 100)  # стоп

# Загрузка модели
print("Загрузка модели...")
model = WhisperModel("base", device="cpu", compute_type="float32")
print("Модель загружена")

# Глобальные переменные
recording = False
audio_buffer = []
pressed_keys = set()

def record_audio():
    """Фоновая запись аудио пока удерживается клавиша"""
    global recording, audio_buffer
    audio_buffer = []
    with sd.InputStream(samplerate=SAMPLE_RATE, channels=CHANNELS, dtype="float32") as stream:
        while recording:
            data, _ = stream.read(1024)
            audio_buffer.append(data)

def process_audio():
    """Обработка и распознавание аудио после завершения записи"""
    global audio_buffer
    if not audio_buffer:
        return

    audio_data = np.concatenate(audio_buffer, axis=0)
    audio_int16 = (audio_data * 32767).astype(np.int16)

    with tempfile.NamedTemporaryFile(suffix=".wav", delete=False) as f:
        wav.write(f.name, SAMPLE_RATE, audio_int16)
        temp_filename = f.name

    print("Распознавание...")
    segments, _ = model.transcribe(temp_filename, language="ru")
    text_result = " ".join(segment.text for segment in segments)
    print("Результат:", text_result)

    kb.write(text_result)
    print("Текст вставлен")

    play_sound_end()

# -----------------------------
# Обработчики клавиш
# -----------------------------
def key_press(key):
    global recording
    pressed_keys.add(key)

    # Вариант 1: Ctrl + Shift
    if keyboard.Key.ctrl_l in pressed_keys and keyboard.Key.shift in pressed_keys:
        if not recording:
            recording = True
            threading.Thread(target=record_audio, daemon=True).start()
            play_sound_start()
            print("Говорите...")

    # Вариант 2: боковая кнопка мыши → F24
    if key == keyboard.Key.f24:
        if not recording:
            recording = True
            threading.Thread(target=record_audio, daemon=True).start()
            play_sound_start()
            print("Говорите...")

def key_release(key):
    global recording

    if key in pressed_keys:
        pressed_keys.remove(key)

    # Остановка записи для Ctrl+Shift
    if recording and (key == keyboard.Key.ctrl_l or key == keyboard.Key.shift):
        recording = False
        print("Запись завершена")
        threading.Thread(target=process_audio, daemon=True).start()

    # Остановка записи для боковой кнопки (F24)
    if recording and key == keyboard.Key.f24:
        recording = False
        print("Запись завершена")
        threading.Thread(target=process_audio, daemon=True).start()

# -----------------------------
# Запуск слушателя
# -----------------------------
with keyboard.Listener(on_press=key_press, on_release=key_release) as listener:
    print("Готово: удерживайте Ctrl+Shift или боковую кнопку мыши для записи")
    listener.join()


```


voice_input.py


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
    winsound.Beep(440, 100)  # 440 Hz, 100 мс

def play_sound_end():
    winsound.Beep(880, 100)  # 880 Hz, 100 мс

# Загрузка модели
print("Загрузка модели...")
model = WhisperModel("base", device="cpu", compute_type="float32")
print("Модель загружена")

# Глобальные переменные
recording = False
audio_buffer = []

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

    # Вставка текста в курсор
    kb.write(text_result)
    print("Текст вставлен в курсор")

    # Проигрываем звук завершения
    play_sound_end()

# Слежение за нажатиями клавиш
pressed_keys = set()

def key_press(key):
    global recording
    pressed_keys.add(key)
    try:
        if keyboard.Key.ctrl_l in pressed_keys and keyboard.Key.shift in pressed_keys and not recording:
            recording = True
            threading.Thread(target=record_audio, daemon=True).start()
            play_sound_start()
            print("Говорите...")
    except AttributeError:
        pass

def key_release(key):
    global recording
    if key in pressed_keys:
        pressed_keys.remove(key)
    # Останавливаем запись и обрабатываем аудио
    if recording and (key == keyboard.Key.ctrl_l or key == keyboard.Key.shift):
        recording = False
        print("Запись завершена")
        threading.Thread(target=process_audio, daemon=True).start()

# Запуск слушателя клавиатуры
with keyboard.Listener(on_press=key_press, on_release=key_release) as listener:
    print("Нажмите Левый Ctrl + Левый Shift, чтобы говорить...")
    listener.join()

```

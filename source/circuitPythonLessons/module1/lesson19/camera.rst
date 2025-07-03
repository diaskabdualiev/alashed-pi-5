============================================================
Урок 19: Работа с камерой Raspberry Pi 📷
============================================================

Теоретическая часть
-------------------
Камера Raspberry Pi – это официальный модуль камеры для Raspberry Pi, который подключается напрямую к плате через CSI (Camera Serial Interface). Существует несколько моделей камер, включая стандартную и NoIR (без ИК-фильтра). Для работы с камерой в CircuitPython мы будем использовать библиотеку `picamera2`, которая предоставляет современный API для работы с камерой на Python.

В этом уроке мы научимся захватывать фотографии, изменять их параметры и создавать простую систему видеонаблюдения с использованием камеры Raspberry Pi.

Необходимые компоненты
----------------------
- Raspberry Pi
- Модуль камеры Raspberry Pi
- (Опционально) Кнопка для запуска съемки

Схема подключения
-----------------
.. figure:: images/camera.webp
   :width: 80%
   :align: center

   **Рис. 1:** Подключение камеры к Raspberry Pi

Установка необходимых библиотек
-------------------------------
Перед запуском кода установите библиотеку picamera2 и OpenCV:

.. code-block:: bash

   sudo apt-get update
   sudo apt-get install -y python3-picamera2
   sudo apt-get install -y python3-opencv

Запуск кода
------------
1. Откройте текстовый редактор или IDE (например, Thonny или VS Code) на Raspberry Pi.
2. Создайте новый файл **camera_circuit.py** в папке `lessons/lesson17/`.
3. Вставьте в файл следующий код:

Код программы
-------------
Файл: `lessons/lesson17/camera_circuit.py`

.. code-block:: python

      import time
      import datetime
      from pathlib import Path
      
      import cv2
      from picamera2 import Picamera2
      
      SAVE_DIR = Path("camera_captures")
      SAVE_DIR.mkdir(exist_ok=True)
      
      picam2 = Picamera2()
      
      PHOTO_CFG = picam2.create_still_configuration(
          main={"size": (1920, 1080), "format": "RGB888"},
          lores={"size": (640, 480), "format": "YUV420"},
      )
      PREVIEW_CFG = picam2.create_preview_configuration(
          main={"size": (640, 480), "format": "RGB888"},
      )
      
      # ─────────── ПАРАМЕТРЫ ДЕТЕКЦИИ ───────────
      AREA_THRESHOLD        = 7_000    # минимальная площадь одного контура (px)
      DIFF_PIXELS_THRESHOLD = 15_000   # минимум активных пикселей после threshold
      CONSEC_MOTION_FRAMES  = 3        # движение должно быть n кадров подряд
      COOLDOWN_FRAMES       = 20       # пауза (кадров) после съёмки
      IDLE_TIMEOUT          = 300      # отсутствие движения (сек) → выход
      
      
      def _ts() -> str:
          return datetime.datetime.now().strftime("%Y%m%d-%H%M%S")
      
      
      # ─────────── ФОТО ──────────
      
      def capture_photo() -> Path:
          try:
              picam2.configure(PHOTO_CFG)
              picam2.start(); time.sleep(2)
              fp = SAVE_DIR / f"photo_{_ts()}.jpg"
              picam2.capture_file(str(fp))
              print("Фото:", fp)
              return fp
          finally:
              picam2.stop()
      
      
      # ─────────── ЭФФЕКТЫ ──────────
      
      def capture_photo_with_effects():
          try:
              picam2.configure(PHOTO_CFG)
              picam2.start(); time.sleep(2)
              img = picam2.capture_array()
              gray, edges = cv2.cvtColor(img, cv2.COLOR_RGB2GRAY), None
              edges = cv2.Canny(gray, 100, 200)
              gfp = SAVE_DIR / f"gray_{_ts()}.jpg"
              efp = SAVE_DIR / f"edges_{_ts()}.jpg"
              cv2.imwrite(str(gfp), gray); cv2.imwrite(str(efp), edges)
              print("Gray:", gfp, "Edges:", efp)
          finally:
              picam2.stop()
      
      
      # ─────────── ДЕТЕКТОР ДВИЖЕНИЯ ──────────
      
      def motion_detection():
          try:
              picam2.configure(PREVIEW_CFG)
              picam2.start(); time.sleep(2)
      
              # базовый фон
              bg = picam2.capture_array()
              bg = cv2.GaussianBlur(cv2.cvtColor(bg, cv2.COLOR_RGB2GRAY), (21, 21), 0).astype("float32")
      
              cooldown = 0
              motion_seq = 0
              last_motion_time = time.time()
      
              print("Детекция движения запущена… (Ctrl+C для выхода)")
              while True:
                  frame = picam2.capture_array()
                  gray  = cv2.GaussianBlur(cv2.cvtColor(frame, cv2.COLOR_RGB2GRAY), (21, 21), 0).astype("float32")
      
                  diff   = cv2.absdiff(bg, gray)
                  thresh = cv2.threshold(cv2.convertScaleAbs(diff), 25, 255, cv2.THRESH_BINARY)[1]
                  thresh = cv2.morphologyEx(thresh, cv2.MORPH_OPEN, None, iterations=2)  # очистка шумов
      
                  motion_pixels = cv2.countNonZero(thresh)
                  contours, _   = cv2.findContours(thresh, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
                  large_contour = any(cv2.contourArea(c) >= AREA_THRESHOLD for c in contours)
      
                  motion_detected = motion_pixels >= DIFF_PIXELS_THRESHOLD and large_contour
      
                  # считаем последовательность кадров с движением
                  if motion_detected:
                      motion_seq += 1
                  else:
                      motion_seq = 0
      
                  if motion_seq >= CONSEC_MOTION_FRAMES and cooldown == 0:
                      fp = SAVE_DIR / f"motion_{_ts()}.jpg"
                      picam2.switch_mode_and_capture_file(PHOTO_CFG, str(fp))
                      print("Движение! Кадр:", fp)
                      picam2.start(); cooldown = COOLDOWN_FRAMES; motion_seq = 0
                      last_motion_time = time.time()
      
                  if cooldown > 0:
                      cooldown -= 1
                  else:
                      cv2.accumulateWeighted(gray, bg, 0.05)
      
                  # завершение при долгом отсутствии движения
                  if time.time() - last_motion_time > IDLE_TIMEOUT:
                      print("Сцена статична > 5 мин — выходим.")
                      break
      
                  time.sleep(0.1)
      
          except KeyboardInterrupt:
              print("\nПрервано пользователем.")
          finally:
              picam2.stop()
      
      
      # ─────────── CLI ──────────
      
      if __name__ == "__main__":
          print("Raspberry Pi Camera Utility")
          print("1 — фото | 2 — эффекты | 3 — motion-detector")
          cmd = input("Выбор: ").strip()
          if cmd == "1":
              capture_photo()
          elif cmd == "2":
              capture_photo_with_effects()
          elif cmd == "3":
              motion_detection()
          else:
              print("Неизвестная команда.")


Разбор кода
------------
- `Picamera2()` – инициализация камеры с использованием современного API Picamera2.
- `create_still_configuration()` – создание конфигурации для фотографий высокого разрешения.
- `create_preview_configuration()` – создание конфигурации для видеопотока с меньшим разрешением.
- `capture_photo()` – функция для захвата фотографии и сохранения ее с временной меткой.
- `capture_photo_with_effects()` – функция для захвата фотографии и применения эффектов с помощью OpenCV:
  - Преобразование в оттенки серого.
  - Применение детектора краев Canny.
- `motion_detection()` – функция для обнаружения движения:
  - Захват фонового изображения.
  - Сравнение текущего кадра с фоном.
  - Обнаружение значительных изменений (контуров).
  - Сохранение кадров с обнаруженным движением.

Ожидаемый результат
-------------------
1. При выборе опции 1 камера делает снимок и сохраняет его в папку `camera_captures`.
2. При выборе опции 2 камера делает снимок, применяет к нему эффекты (черно-белый и выделение краев) и сохраняет результаты.
3. При выборе опции 3 камера запускает систему обнаружения движения на 30 секунд, которая автоматически сохраняет кадры при обнаружении движения.

.. note::
   Для наилучших результатов рекомендуется использовать хорошее освещение. Параметры обнаружения движения могут потребовать настройки в зависимости от условий окружающей среды. Обратите внимание, что обработка изображений с помощью OpenCV может быть ресурсоемкой на Raspberry Pi.

Завершение работы
-----------------
По окончании работы программы или при нажатии **Ctrl + C** камера автоматически останавливается.

Поздравляем! 🎉 Вы успешно научились работать с камерой Raspberry Pi с помощью CircuitPython и OpenCV! Теперь вы можете использовать камеру для создания проектов с распознаванием образов, видеонаблюдением и других интересных приложений компьютерного зрения.

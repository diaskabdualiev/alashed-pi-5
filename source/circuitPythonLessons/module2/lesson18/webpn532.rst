========================================================================================================================
Веб-интерфейс для NFC-считывателя 
========================================================================================================================

Теоретическая часть
--------------------------------------
В этом уроке мы создадим веб-приложение для NFC-считывателя на базе Raspberry Pi. Такая система позволит удаленно отслеживать считывание NFC-меток и карт через веб-браузер. Мы объединим технологии работы с NFC и веб-разработки, сделав информацию с NFC-считывателя доступной через интернет.

Ключевые технологии, которые будут использованы:
- **PN532** - популярный чип для работы с NFC/RFID
- **CircuitPython** с библиотекой **adafruit_pn532** для работы с NFC-считывателем
- **Flask** - легковесный веб-фреймворк для создания веб-приложения
- **Многопоточность** (threading) для одновременной работы NFC-считывателя и веб-сервера
- **HTML/CSS/JavaScript** для создания интерактивного интерфейса

Необходимые компоненты
-----------------------------------------
- Raspberry Pi
- NFC-считыватель PN532
- NFC-метки или карты для тестирования
- Соединительные провода

Схема подключения
-------------------------------------------------------

.. figure:: images/image.png
   :width: 80%
   :align: center

NFC-считыватель PN532 подключается к Raspberry Pi по интерфейсу I2C:

Установка необходимых библиотек
---------------------------------------------------------------------
Для начала работы необходимо установить все требуемые библиотеки:

.. code-block:: bash

   # Обновление списка пакетов
   sudo apt-get update
   
   # Установка I2C утилит
   sudo apt-get install -y python3-pip i2c-tools
   
   # Включение I2C в Raspberry Pi (если еще не включен)
   sudo raspi-config
   # Выберите: Interfacing Options → I2C → Yes
   
   # Установка библиотек Python
   pip3 install flask
   pip3 install adafruit-blinka
   pip3 install adafruit-circuitpython-pn532

Проверка подключения NFC-считывателя:

.. code-block:: bash

   sudo i2cdetect -y 1

В выводе команды должен присутствовать адрес устройства (обычно 0x24), что подтверждает успешное подключение считывателя.

Структура проекта
--------------------------------------------------------------------------------------
Создадим следующую структуру файлов:

.. code-block:: bash

   nfc_web_app/
   ├── app.py             # Серверная часть на Flask с NFC-считывателем
   └── templates/
       └── index.html     # Веб-интерфейс

Серверная часть (app.py)
---------------------------------------------------------------------------------------------
Создайте файл `app.py` с следующим содержимым:

.. code-block:: python

   from flask import Flask, render_template, jsonify
   import board
   import busio
   import digitalio
   import threading
   import time
   from adafruit_pn532.i2c import PN532_I2C

   app = Flask(__name__)

   # Глобальные переменные для хранения данных с NFC
   last_uid = None
   is_running = True

   # Инициализация PN532
   def init_pn532():
       try:
           # I2C-шина и пин reset
           i2c = busio.I2C(board.SCL, board.SDA)
           reset = digitalio.DigitalInOut(board.D25)
           
           # Создаем объект PN532
           pn532 = PN532_I2C(i2c, debug=False, reset=reset)
           
           # Выводим информацию о версии прошивки
           ic, ver, rev, support = pn532.firmware_version
           print(f"PN532 v{ver}.{rev} — IC 0x{ic:x}")
           
           # Включаем чтение карт
           pn532.SAM_configuration()
           
           return pn532
       except Exception as e:
           print(f"Ошибка инициализации PN532: {e}")
           return None

   # Функция для считывания NFC в отдельном потоке
   def read_nfc():
       global last_uid, is_running
       
       # Инициализируем PN532
       pn532 = init_pn532()
       if not pn532:
           print("Не удалось инициализировать PN532, завершение работы")
           return
       
       print("Поднесите NFC-метку...")
       
       # Основной цикл сканирования
       while is_running:
           try:
               # Пытаемся считать NFC метку
               uid = pn532.read_passive_target(timeout=0.5)
               
               if uid:
                   uid_hex = uid.hex()
                   print(f"Найдена карта, UID: {uid_hex}")
                   last_uid = uid_hex
                   
                   # Небольшая задержка, чтобы избежать повторного считывания
                   time.sleep(0.5)
           except Exception as e:
               print(f"Ошибка при считывании NFC: {e}")
               time.sleep(1)

   # Запускаем поток считывания NFC
   nfc_thread = threading.Thread(target=read_nfc)
   nfc_thread.daemon = True
   nfc_thread.start()

   @app.route('/')
   def index():
       return render_template('index.html')

   @app.route('/get_uid')
   def get_uid():
       return jsonify({'uid': last_uid})

   if __name__ == '__main__':
       try:
           print("NFC веб-приложение запущено")
           app.run(host='0.0.0.0', port=5000, debug=True, use_reloader=False)
       except KeyboardInterrupt:
           print("Программа остановлена")
           is_running = False

Веб-интерфейс (index.html)
-----------------------------------------------------------------------------------------------
Создайте директорию `templates` и в ней файл `index.html` с следующим содержимым:

.. code-block:: html

   <!DOCTYPE html>
   <html lang="ru">
   <head>
       <meta charset="UTF-8">
       <meta name="viewport" content="width=device-width, initial-scale=1.0">
       <title>NFC Считыватель</title>
       <style>
           body {
               font-family: Arial, sans-serif;
               max-width: 600px;
               margin: 0 auto;
               padding: 20px;
               background-color: #f5f5f5;
           }
           h1 {
               color: #333;
               text-align: center;
           }
           .container {
               background-color: white;
               border-radius: 10px;
               padding: 20px;
               box-shadow: 0 2px 10px rgba(0,0,0,0.1);
               margin-top: 20px;
           }
           .uid-card {
               border: 1px solid #ddd;
               border-radius: 10px;
               padding: 20px;
               margin: 20px 0;
               background-color: #f9f9f9;
               text-align: center;
           }
           .uid-text {
               font-size: 22px;
               font-weight: bold;
               margin: 15px 0;
               font-family: monospace;
           }
           .status {
               text-align: center;
               font-style: italic;
               color: #666;
               margin: 10px 0;
           }
           .loading {
               display: inline-block;
               width: 20px;
               height: 20px;
               border: 3px solid #f3f3f3;
               border-top: 3px solid #3498db;
               border-radius: 50%;
               animation: spin 1s linear infinite;
               margin-right: 10px;
               vertical-align: middle;
           }
           @keyframes spin {
               0% { transform: rotate(0deg); }
               100% { transform: rotate(360deg); }
           }
           .last-update {
               text-align: right;
               font-size: 12px;
               color: #999;
               margin-top: 20px;
           }
           .nfc-icon {
               width: 80px;
               height: 80px;
               margin: 0 auto;
               display: block;
               background-color: #3498db;
               border-radius: 50%;
               position: relative;
           }
           .nfc-icon:before {
               content: "";
               position: absolute;
               top: 20%;
               left: 20%;
               width: 60%;
               height: 60%;
               border: 4px solid white;
               border-radius: 50%;
               box-sizing: border-box;
           }
       </style>
   </head>
   <body>
       <h1>NFC Считыватель</h1>
       
       <div class="container">
           <div class="nfc-icon"></div>
           
           <div class="status" id="status">
               <span class="loading"></span> Ожидание NFC метки...
           </div>
           
           <div class="uid-card">
               <h2>Последний считанный UID:</h2>
               <div class="uid-text" id="uid-display">Нет данных</div>
           </div>
           
           <div class="last-update" id="last-update">
               Последнее обновление: Никогда
           </div>
       </div>

       <script>
           // Функция для получения текущего времени в формате ЧЧ:ММ:СС
           function getCurrentTime() {
               const now = new Date();
               return now.toLocaleTimeString();
           }
           
           // Функция для получения UID с сервера
           function getUID() {
               fetch('/get_uid')
                   .then(response => response.json())
                   .then(data => {
                       const uidDisplay = document.getElementById('uid-display');
                       const status = document.getElementById('status');
                       const lastUpdate = document.getElementById('last-update');
                       
                       // Обновляем время последнего обновления
                       lastUpdate.textContent = 'Последнее обновление: ' + getCurrentTime();
                       
                       if (data.uid) {
                           // Если UID получен
                           uidDisplay.textContent = data.uid;
                           status.innerHTML = '<span style="color: green;">✓</span> NFC метка обнаружена';
                       } else {
                           // Если UID не получен
                           uidDisplay.textContent = 'Нет данных';
                           status.innerHTML = '<span class="loading"></span> Ожидание NFC метки...';
                       }
                   })
                   .catch(error => {
                       console.error('Ошибка:', error);
                       const status = document.getElementById('status');
                       status.innerHTML = '<span style="color: red;">✗</span> Ошибка связи с сервером';
                   });
           }
           
           // Получаем UID при загрузке страницы
           getUID();
           
           // Обновляем UID каждые 3 секунды
           setInterval(getUID, 3000);
           
           // Перезагружаем страницу каждую минуту
           setInterval(function() {
               location.reload();
           }, 60000);
       </script>
   </body>
   </html>

Запуск приложения
--------------------------------------------------------------------------------------
1. Создайте необходимую структуру директорий и файлы:

   .. code-block:: bash

      mkdir -p nfc_web_app/templates
      cd nfc_web_app
      # Создайте файлы app.py и templates/index.html с указанным выше содержимым

2. Запустите приложение:

   .. code-block:: bash

      python3 app.py

3. Откройте веб-браузер и перейдите по адресу:

   .. code-block:: bash

      http://<IP-адрес_Raspberry_Pi>:5000

Разбор кода
--------------------------------------------------------------------------------

### Серверная часть (app.py)

Рассмотрим ключевые элементы серверной части приложения:

**Инициализация и глобальные переменные:**

В начале кода мы создаем Flask-приложение и определяем глобальные переменные:

.. code-block:: python

   app = Flask(__name__)

   # Глобальные переменные для хранения данных с NFC
   last_uid = None
   is_running = True

- `last_uid` хранит последний считанный UID карты
- `is_running` управляет работой потока считывания

**Функция инициализации NFC-считывателя:**

Функция `init_pn532()` отвечает за инициализацию и настройку NFC-считывателя:

.. code-block:: python

   def init_pn532():
       try:
           # I2C-шина и пин reset
           i2c = busio.I2C(board.SCL, board.SDA)
           reset = digitalio.DigitalInOut(board.D25)
           
           # Создаем объект PN532
           pn532 = PN532_I2C(i2c, debug=False, reset=reset)
           
           # Выводим информацию о версии прошивки
           ic, ver, rev, support = pn532.firmware_version
           print(f"PN532 v{ver}.{rev} — IC 0x{ic:x}")
           
           # Включаем чтение карт
           pn532.SAM_configuration()
           
           return pn532
       except Exception as e:
           print(f"Ошибка инициализации PN532: {e}")
           return None

В этой функции:
1. Инициализируем I2C интерфейс для связи с PN532
2. Настраиваем пин сброса (reset)
3. Создаем объект PN532 и проверяем версию прошивки
4. Настраиваем Security Access Module (SAM) для работы с картами
5. Возвращаем инициализированный объект или None в случае ошибки

**Функция считывания NFC в отдельном потоке:**

.. code-block:: python

   def read_nfc():
       global last_uid, is_running
       
       # Инициализируем PN532
       pn532 = init_pn532()
       if not pn532:
           print("Не удалось инициализировать PN532, завершение работы")
           return
       
       print("Поднесите NFC-метку...")
       
       # Основной цикл сканирования
       while is_running:
           try:
               # Пытаемся считать NFC метку
               uid = pn532.read_passive_target(timeout=0.5)
               
               if uid:
                   uid_hex = uid.hex()
                   print(f"Найдена карта, UID: {uid_hex}")
                   last_uid = uid_hex
                   
                   # Небольшая задержка, чтобы избежать повторного считывания
                   time.sleep(0.5)
           except Exception as e:
               print(f"Ошибка при считывании NFC: {e}")
               time.sleep(1)

Функция `read_nfc()` запускается в отдельном потоке и выполняет:
1. Инициализацию NFC-считывателя
2. Постоянный опрос NFC-считывателя на наличие карты
3. При обнаружении карты сохраняет её UID в глобальную переменную

**Запуск потока считывания:**

.. code-block:: python

   nfc_thread = threading.Thread(target=read_nfc)
   nfc_thread.daemon = True
   nfc_thread.start()

Здесь мы:
1. Создаем новый поток, который будет выполнять функцию `read_nfc()`
2. Устанавливаем флаг `daemon=True`, чтобы поток автоматически завершался при выходе из основной программы
3. Запускаем поток

**Маршруты Flask:**

.. code-block:: python

   @app.route('/')
   def index():
       return render_template('index.html')

   @app.route('/get_uid')
   def get_uid():
       return jsonify({'uid': last_uid})

Создаем два маршрута:
1. `/` - для отображения веб-интерфейса
2. `/get_uid` - API-эндпоинт, который возвращает последний считанный UID в формате JSON

**Запуск веб-сервера:**

.. code-block:: python

   if __name__ == '__main__':
       try:
           print("NFC веб-приложение запущено")
           app.run(host='0.0.0.0', port=5000, debug=True, use_reloader=False)
       except KeyboardInterrupt:
           print("Программа остановлена")
           is_running = False

При запуске:
1. Выводим сообщение о запуске приложения
2. Запускаем Flask-сервер на всех интерфейсах (0.0.0.0)
3. Отключаем автоматическую перезагрузку (`use_reloader=False`), чтобы избежать проблем с потоками
4. При нажатии Ctrl+C устанавливаем `is_running=False` для корректного завершения потока считывания

### Веб-интерфейс (index.html)

Рассмотрим основные элементы веб-интерфейса:

**HTML-структура:**

HTML-документ содержит несколько ключевых элементов:
1. Заголовок страницы
2. Контейнер с иконкой NFC
3. Индикатор статуса
4. Карточку для отображения UID
5. Строку с временем последнего обновления

**CSS-стили:**

CSS-стили отвечают за внешний вид интерфейса:
- Адаптивный дизайн с максимальной шириной 600px
- Карточка с тенью для основного контента
- Анимированный индикатор загрузки
- Стилизованная иконка NFC
- Различные стили для статусов и текста

**JavaScript для взаимодействия с сервером:**

JavaScript-код выполняет следующие функции:
1. Получает текущее время для отображения времени обновления
2. Делает AJAX-запросы к API `/get_uid` для получения последнего UID
3. Обновляет интерфейс в зависимости от полученных данных
4. Настраивает автоматическое обновление каждые 3 секунды
5. Перезагружает страницу каждую минуту для обновления состояния

Ожидаемый результат
----------------------------------------------------------------------------------------

При запуске приложения и открытии веб-интерфейса вы увидите:
1. Страницу с иконкой NFC и статусом "Ожидание NFC метки..."
2. При поднесении NFC-карты или метки к считывателю, интерфейс обновится:

   - Статус изменится на "NFC метка обнаружена"
   - Отобразится UID карты
   - Обновится время последнего обновления

Интерфейс будет автоматически обновляться каждые 3 секунды, проверяя наличие новых данных с NFC-считывателя.

Возможные расширения проекта
-----------------------------------------------------------------------------------------------

Созданное приложение можно расширить несколькими способами:

1. **База данных зарегистрированных карт**:
   - Добавить SQLite или MySQL для хранения UID карт
   - Реализовать регистрацию новых карт через интерфейс
   - Показывать информацию о владельце карты при считывании

2. **Система контроля доступа**:
   - Добавить управление электронным замком через реле
   - Логирование всех попыток доступа с временной меткой
   - Разные уровни доступа для разных карт

3. **Мобильные уведомления**:
   - Отправлять push-уведомления при считывании определенных карт
   - Интеграция с Telegram, Email или SMS для оповещений

4. **Расширенная аналитика**:
   - Графики использования в течение дня/недели
   - Статистика посещаемости
   - Экспорт данных в различных форматах

5. **Защищенный доступ к интерфейсу**:
   - Авторизация для доступа к веб-интерфейсу
   - HTTPS для безопасного соединения
   - Различные роли пользователей (администратор, наблюдатель)

Советы по отладке
-------------------------------------------------------------------------------------

1. **Проблемы с I2C**:
   - Проверьте подключение проводов
   - Выполните `sudo i2cdetect -y 1` для проверки наличия устройства
   - Убедитесь, что I2C включен в `raspi-config`

2. **Не считываются карты**:
   - Проверьте расстояние между картой и считывателем (оптимально 1-3 см)
   - Убедитесь, что карта совместима с PN532 (ISO 14443A/MIFARE)
   - Проверьте питание считывателя (должно быть стабильным)

3. **Проблемы с веб-интерфейсом**:
   - Проверьте логи Flask на наличие ошибок
   - Используйте инструменты разработчика в браузере (F12) для проверки сетевых запросов
   - Если страница не обновляется, проверьте JavaScript-консоль на наличие ошибок

Заключение
------------------------------------------------------------------------------

В этом уроке мы создали полноценное веб-приложение для работы с NFC-считывателем. Мы научились:
- Подключать и инициализировать NFC-считыватель PN532
- Создавать многопоточное приложение для параллельной работы NFC и веб-сервера
- Разрабатывать API для обмена данными между сервером и клиентом
- Создавать интерактивный веб-интерфейс с автоматическим обновлением

Такое приложение может служить основой для различных проектов, связанных с NFC-технологиями: от простых систем идентификации до комплексных решений контроля доступа.

# Homework Helper Bot

Telegram-бот для помощи студентам British Management University (BMU) в Узбекистане с выполнением домашних заданий (ДЗ). Бот принимает задания, оценивает сложность и цену с помощью ИИ (Gemini), обрабатывает платежи (на старте — ручные), генерирует решения и отслеживает статусы.

## Назначение
- **Целевая аудитория**: Студенты BMU, 18–25 лет.
- **Ключевые функции**: Прием ДЗ (текст/файлы/фото), ИИ-оценка, подтверждение, оплата (тестовая карта), ручная/автоматическая генерация решений, статусы заказов, статистика.
- **Этапы**: Ручной MVP (Этап 1) → Автоматизация с n8n (Этап 2) → Масштаб с дашбордом (Фаза 3).
- **Язык**: Русский (с опцией английского в будущем).

Подробное ТЗ: [Ссылка на ТЗ, если есть].

## Технический стек
- **Бэкенд**: Python 3.12 + python-telegram-bot v20+.
- **БД**: SQLite (homework_bot.db).
- **ИИ**: Google Gemini API (gemini-1.5-flash).
- **Генерация файлов**: python-docx + reportlab.
- **Автоматизация**: n8n (self-host).
- **Деплой**: Heroku (бот), Render.com (n8n).
- **Версионирование**: Git/GitHub.

## Установка и запуск (локально)

1. **Клонируй репозиторий**:
   ```
   git clone https://github.com/your-username/homework-helper-bot.git
   cd homework-helper-bot
   ```

2. **Создай виртуальное окружение** (рекомендуется):
   ```
   python -m venv venv
   source venv/bin/activate  # На Windows: venv\Scripts\activate
   ```

3. **Установи зависимости**:
   ```
   pip install -r requirements.txt
   ```

4. **Настрой секреты**:
   - Создай файл `.env` в корне проекта (добавь в `.gitignore`):
     ```
     TOKEN=your_telegram_bot_token  # От @BotFather
     GEMINI_KEY=your_gemini_api_key  # От ai.google.dev
     ADMIN_ID=your_telegram_user_id  # Твой ID как админ
     N8N_WEBHOOK_URL=https://your-n8n.onrender.com/webhook/  # Для Этапа 2
     ```
   - Получи ключи:
     - Telegram: @BotFather → /newbot.
     - Gemini: ai.google.dev → API keys (free tier).

5. **Инициализируй БД**:
   ```
   python init_db.py
   ```
   Это создаст `homework_bot.db` с таблицами: users, orders, logs.

6. **Запусти бота** (для теста — polling):
   ```
   python bot.py
   ```
   - Для продакшена: Установи webhook в bot.py (см. ТЗ 8.2).

7. **Настрой n8n (для Этапа 2)**:
   - Локально: `docker run -it --rm --name n8n -p 5678:5678 -v ~/.n8n:/home/node/.n8n n8nio/n8n`.
   - Импортируй workflow JSON (пример в репозитории или n8n community).
   - Деплой: Render.com (см. ТЗ 8.3).

## Использование
- **Пользователь**:
  - `/start` — Регистрация (введите имя).
  - `/dz` — Отправьте ДЗ (текст, файл PDF/DOCX/TXT, фото для OCR).
  - Подтвердите оценку (кнопки Да/Нет), оплатите на тестовую карту: 4557 4304 0205 3431 (Uzcard).
  - `/my_orders` — Список активных заказов.
- **Админ** (только ADMIN_ID):
  - `/admin_confirm {order_id}` — Подтвердить оплату.
  - `/send_dz {order_id}` — Отправить готовое ДЗ (ручное).
  - `/stats` — Статистика (заказы, доход, отзывы).
- **Автоматизация**: После подтверждения — webhook в n8n для генерации DOCX/PDF.

## Промпты для Gemini
- **Анализ ДЗ** (функция `analyze_dz`):
  ```
  Анализируй ДЗ: {content}. Оцени: complexity (low/medium/high), deadline (12-24ч, по умолчанию 18ч), price (20000-100000 UZS). Верни только JSON: {"complexity": "...", "deadline": "...", "price": 50000}.
  ```
- **Генерация решения** (в n8n):
  ```
  Используй шаблон [ссылка на пример ДЗ в Google Drive]. Реши задание: {dz_content}. Формат: Титульный лист (British Management University, студент: {user_name}, дата: {current_date}), раздел "Решение" с шагами, "Источники" в конце. Верни текст для DOCX.
  ```
- **OCR для фото**:
  ```
  Извлеки текст из изображения: [image_description]. Верни чистый текст ДЗ.
  ```

## Схема БД (SQLite: homework_bot.db)
Создана в `init_db.py`. Запросите в DB Browser for SQLite для просмотра.

```sql
-- Таблица пользователей
CREATE TABLE users (
    user_id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_active TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица заказов
CREATE TABLE orders (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    dz_file_path TEXT,  -- Путь к файлу ДЗ (/files/{user_id}_{timestamp}.*)
    dz_content TEXT,    -- Текст/OCR из файла
    complexity TEXT,    -- 'low'/'medium'/'high'
    deadline TEXT,      -- '18ч'
    price INTEGER,      -- UZS
    status TEXT DEFAULT 'new',  -- 'new', 'waiting_payment', 'accepted', 'completed', 'cancelled', 'rejected'
    payment_status TEXT DEFAULT 'pending',  -- 'pending', 'paid', 'failed'
    order_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,
    review_score INTEGER,  -- 1-5
    FOREIGN KEY (user_id) REFERENCES users (user_id)
);

-- Таблица логов
CREATE TABLE logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    order_id INTEGER,
    event TEXT,  -- 'dz_received', 'payment_confirmed', etc.
    details TEXT,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (order_id) REFERENCES orders (id)
);

-- Индексы
CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
```

**Операции**:
- Вставка: `INSERT INTO users (user_id, name) VALUES (?, ?);`
- Запросы: `SELECT * FROM orders WHERE user_id = ? AND status != 'completed';`
- Обновления: `UPDATE orders SET status = 'accepted' WHERE id = ?;`

## Деплой
- **Heroku** (бот):
  ```
  heroku create homework-helper-bot
  git push heroku main
  heroku ps:scale worker=1
  heroku config:set WEBHOOK_URL=https://your-app.herokuapp.com/
  ```
- **n8n на Render**: Connect GitHub → Web Service → Docker (n8nio/n8n) → Set vars (N8N_BASIC_AUTH_ACTIVE=true).
- **Бэкап**: n8n cron для dump БД в Google Drive.

## Тестирование
- Unit-тесты: `pytest` (тесты в `tests/`).
- Интеграционные: Симулируйте флоу в Telegram (тестовые аккаунты).
- Load-test: 10 concurrent users.

## Риски и рекомендации
- **Лимит Gemini**: 25 запросов/день — fallback на ручную оценку (queue в bot.py).
- **Безопасность**: Анонимизация (user_id в логах), тестовая карта.
- **Юридическое**: В /start добавьте: "Это помощь в обучении, не замена самостоятельной работе."
- **Улучшения**: A/B-тест цен, миграция на PostgreSQL при >100 заказов/мес.

## Контакты
За вопросы — @your_telegram. Общий срок разработки: 7–12 дней. Бюджет: 0 UZS. 🚀

*Версия: 1.0 (MVP, 03.10.2025)*
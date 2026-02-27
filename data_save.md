# 2. Хранение данных

**Цель**  
Надёжно хранить информацию о пользователях, фильмах, оценках и кэшированных постерах, чтобы бот мог быстро показывать карточки фильмов и строить рекомендации без лишних запросов к TMDb.

**Основные таблицы**

1. **users** — пользователи бота и их связь с TMDb  
   - telegram_user_id (PK)  
   - tmdb_account_id  
   - tmdb_session_id (v3, для чтения и возможной записи оценок)  
   - sync_status (0 = не подключён, 1 = в очереди импорта, 2 = успешно, -1 = ошибка)  
   - last_sync_at (UNIX timestamp)  
   - ratings_count (количество импортированных оценок)

2. **movies** — метаданные фильмов из TMDb + статус постеров  
   - id (PK, TMDb movie id)  
   - title, original_title, overview, release_date, year (вычисляемый), runtime, popularity, vote_average, vote_count  
   - poster_path (путь TMDb)  
   - poster_status (-1 = нет постера, 0 = ждёт скачивания, 1 = скачан локально, 2 = загружен в Telegram)  
   - telegram_file_id (главное поле — file_id для отправки в Telegram)  
   - width, height (размеры фото)  
   - local_path (временно, удаляется после загрузки в TG)

3. **user_movie_ratings** — оценки пользователей  
   - telegram_user_id + movie_id (составной PK)  
   - tmdb_rating (оригинал 0.5–10.0 от TMDb)  
   - normalized_rating (1–10 или 1–5 — для будущих расчётов сходства)  
   - rated_at (UNIX timestamp)  
   - source ('tmdb' / 'manual')

**Кэширование постеров (Telegram как CDN)**  
- Постеры скачиваются с `https://image.tmdb.org/t/p/w500{poster_path}`  
- Сохраняются локально временно  
- Загружаются в приватный канал через `send_photo` / `send_file`  
- Сохраняется `telegram_file_id` → локальный файл удаляется  
- При показе карточки фильма бот использует `file_id` → мгновенная отдача без обращения к TMDb

**Автодозагрузка отсутствующих данных**  
- При импорте оценок проверяем наличие фильма в таблице `movies`  
- Если фильма нет → запрос `/movie/{id}?language=ru-RU` → сохраняем метаданные  
- Если есть poster_path → ставим poster_status = 0 → отдельный worker загружает постер и отправляет в Telegram  
- Worker-ы:  
  - один скачивает постеры с TMDb  
  - второй загружает скачанные в Telegram-канал и получает file_id

**Индексы (ключевые)**  
- user_movie_ratings: по telegram_user_id, по movie_id  
- movies: по poster_status (для worker-ов)  
- movies: по telegram_file_id (если поиск по file_id)

**Особенности реализации**  
- Данные из TMDb кэшируются навсегда (или с редким refresh метаданных раз в 3–6 месяцев)  
- Локальные файлы постеров удаляются после успешной загрузки в Telegram  
- session_id хранится открыто (read+write токен), но в будущем можно добавить шифрование

# 05. n8n Workflow A — Intake (Rocket.Chat → Asana → Rocket.Chat)

Цель workflow A:

- принять событие “новое сообщение” из Rocket.Chat
- избежать дублей (дедуп по `message_id + thread_id`)
- создать задачу в Asana в проекте Inbox
- ответить в тред Rocket.Chat ссылкой на задачу

---

## 1) Подготовка (что должно быть уже сделано)

- Rocket.Chat настроен по `03_rocketchat_setup.md`
- Asana проект и поля настроены по `02_asana_setup.md`
- n8n поднят и есть credentials по `04_n8n_setup.md`
- Вы знаете:
  - Asana `workspace_gid`
  - Asana `project_gid` для “Inbox from Rocket.Chat”
  - (опц.) `section_gid` для секции `New`
  - `custom_field_gid` для поля `Source`

### Как быстро найти GID в Asana (без инструментов разработчика)
Обычно GID виден в URL в браузере:

- проект: `.../0/<project_gid>/list`
- задача: `.../0/<project_gid>/<task_gid>`

Если GID в URL не виден (зависит от интерфейса/домена), используйте самый простой путь:

- попросите администратора Asana дать вам “ID проекта/поля”
- либо найдите его через встроенный “API explorer/интеграции” (если доступно)

---

## 2) Создайте новый workflow в n8n

Название: **Workflow A — Rocket.Chat Intake**

---

## 3) Узлы (nodes) и логика — пошагово

Ниже описан “скелет” workflow. Названия узлов можно делать любыми, но лучше близкими к этим.

### Узел 1: Webhook Trigger (вход от Rocket.Chat)
- Тип: Webhook
- Метод: POST
- Path: например `/rc-intake`
- Response: “Respond immediately” (рекомендуемо), чтобы Rocket.Chat быстро получал 200 OK

**Важно:** сохраните выданный URL — он нужен в настройке Outgoing Webhook в Rocket.Chat.

### Узел 2: Validate source / secret (защита)
Цель: убедиться, что запрос пришёл именно от Rocket.Chat.

Реализация без кода:

- извлеките из payload поле “token/secret” (если Rocket.Chat его присылает)
- сравните с ожидаемым значением (IF node)
- если не совпало: завершить workflow (или вернуть 403/сообщение “unauthorized”)

Если Rocket.Chat не отправляет токен в payload — используйте:

- Basic Auth на webhook (если доступно)
- или IP allowlist на уровне сети

### Узел 3: Extract fields (нормализация данных)
Цель: привести входные данные к единому формату, чтобы дальше работать одинаково.

Нужно получить:

- `message_id`
- `room_id`
- `thread_id` (если сообщение в треде — берём id треда; если нет — используем `message_id` как thread_id)
- `author`
- `text`
- `timestamp`

Практика:

- используйте узел типа “Set” (или аналог) и заполните поля выражениями из входного JSON
- если структура payload отличается — поправьте маппинг под вашу версию Rocket.Chat

### Узел 4: Build dedup_key
Создайте поле:

- `dedup_key = message_id + ":" + thread_id`

### Узел 5: Data Store → Get (проверка дубля)
- Store/Collection: `rc_dedup`
- Key: `dedup_key`

### Узел 6: IF “already exists?”
Разветвление:

- **Да, запись в `rc_dedup` найдена** → “не создаём задачу”, просто возвращаем ссылку в тред.
- **Нет, записи нет** → создаём задачу в Asana.

---

## 4) Ветка A: дедуп сработал (задача уже есть)

### Узел A1: Prepare Asana link
Возьмите `asana_task_gid` из найденной записи Data Store и сформируйте ссылку на задачу.

Рекомендуемый формат ссылки (чаще всего работает):

- `https://app.asana.com/0/<project_gid>/<task_gid>`

Если в вашей организации другой домен/регион — возьмите ссылку из браузера Asana и подставляйте только `task_gid`.

### Узел A2: Rocket.Chat → Post message in thread
Отправьте сообщение в Rocket.Chat:

- канал/комната: `room_id`
- тред: `thread_id`
- текст: “Задача уже была создана ранее: <asana_link>”

Технически это делается через Rocket.Chat REST API (в n8n — HTTP Request node с credentials Rocket.Chat).

Важно: в разных версиях Rocket.Chat поля для “ответа в тред” могут называться по‑разному (например, `tmid`). Проверьте на тестовом сообщении и зафиксируйте рабочую схему.

---

## 5) Ветка B: создаём новую задачу в Asana

### Узел B1: Build task payload (name + description)
Сформируйте:

- **name**:
  - вариант 1: первая строка сообщения (до перевода строки), обрезать до разумной длины
  - вариант 2 (если текст короткий/неформатный): “Запрос из Rocket.Chat”
- **description/notes** (рекомендуемый шаблон):
  - “Источник: Rocket.Chat”
  - “Автор: <author>”
  - “Время: <timestamp>”
  - “Ссылка на тред: <rocket_chat_link>” (если вы можете её сформировать)
  - “Текст запроса:” далее полный `text`
  - “Служебно: message_id=<...>, thread_id=<...>”

### Узел B2: Asana → Create Task
Сделайте запрос “Create task” к Asana API (в n8n — Asana node или HTTP Request node с Asana credentials).

Минимально передайте:

- workspace: `workspace_gid` (если требуется)
- projects: добавить `project_gid` (“Inbox from Rocket.Chat”)
- name / notes
- custom_fields:
  - `Rocket.Chat Link` = (ссылка на тред/сообщение, или хотя бы `thread_id`)
  - `Source` = `Rocket.Chat`

#### Важное про поле `Source` (Dropdown)
Asana API обычно принимает **не текст “Rocket.Chat”**, а внутренний ID (GID) опции поля.

Как получить ID опции “Rocket.Chat” без программирования:

1) В n8n создайте временный узел HTTP Request “Get custom field” (GET по custom_field_gid).
2) Посмотрите в ответе список `enum_options` и найдите опцию с именем “Rocket.Chat”.
3) Скопируйте её GID и используйте его в workflow для заполнения поля `Source`.

Если это слишком сложно на старте — временно сделайте `Source` текстовым полем (Text) и заполняйте строкой `Rocket.Chat`.

### Узел B3 (опционально): Put task into section `New`
Если вы хотите, чтобы новые задачи сразу попадали в секцию `New`:

- добавьте шаг “Add task to section” (Asana API) с `section_gid`

### Узел B4: Data Store → Set (сохранить дедуп и обратный индекс)
Запишите 2 записи:

1) `rc_dedup`:
   - key: `dedup_key`
   - value: `{ asana_task_gid, created_at, rc_room_id, rc_thread_id }`

2) `asana_to_rc`:
   - key: `asana_task_gid`
   - value: `{ rc_room_id, rc_thread_id, dedup_key }`

### Узел B5: Rocket.Chat → Reply in thread (ссылка на Asana)
Отправьте в тред:

- “Черновик задачи создан в Asana: <asana_link>”

Ссылку берите из:

- сформированной строки `asana_link`, или
- из ответа Asana (если он возвращает `permalink_url`)

---

## 6) Обработка ошибок (рекомендуемо, но минимально)

Минимальная политика:

- если Asana API вернул ошибку при создании задачи → отправить одно сообщение в `#pm-alerts`:
  - “Intake: не удалось создать задачу”
  - причина (код/краткий текст)
  - данные: author, время, первые 100–200 символов текста

---

## 7) Проверка workflow A (тест‑чек‑лист)

1) Напишите тестовый запрос в `#task-requests`.
2) Убедитесь:
   - в Asana создана задача в “Inbox from Rocket.Chat”
   - в задаче заполнены `Source` и `Rocket.Chat Link` (или хотя бы link/ids)
   - бот ответил в тред ссылкой на задачу
3) Отправьте *тот же* webhook повторно (или повторите сообщение, если Rocket.Chat может продублировать событие).
4) Убедитесь:
   - новая задача **не** создана
   - бот прислал “уже создано ранее” + ссылка


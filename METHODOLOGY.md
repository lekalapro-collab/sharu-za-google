# Методология: Как собирать работающие n8n workflows с первого раза

---

## Главный принцип: Инкрементальная разработка

```
❌ НЕПРАВИЛЬНО: Собрать весь workflow → Запустить → 10 ошибок → Искать где

✅ ПРАВИЛЬНО: 1 нода → Тест → Работает → Следующая нода → Тест → ...
```

---

## 7 правил безошибочной разработки

### 1. Один шаг — один тест

```
Добавил ноду → Нажал "Test step" → Проверил output → Только потом следующая
```

**Пример:**
```
1. Telegram Trigger → Test → Вижу JSON сообщения ✓
2. + Set node → Test → Данные извлекаются ✓
3. + Google Drive → Test → Файл загружается ✓
```

### 2. Сначала hardcode, потом expressions

```
❌ Сразу: chatId = {{ $json.message.chat.id }}
✅ Сначала: chatId = 530405361 (твой ID)
   Работает? → Меняем на expression
```

**Почему:** Если не работает с hardcode — проблема в ноде. Если работает с hardcode, но не с expression — проблема в выражении.

### 3. Проверяй данные ПЕРЕД нодой

Добавь временный **Set node** чтобы увидеть что приходит:

```javascript
// Добавь в Set node перед проблемной нодой
{
  "debug_input": {{ JSON.stringify($json) }},
  "debug_keys": {{ Object.keys($json) }}
}
```

### 4. Используй Pin Data

После успешного теста — **закрепи данные**:
1. Выполни ноду
2. Нажми на output
3. "Pin data"

Теперь можешь тестировать следующие ноды без повторного запуска триггера.

### 5. Expression Editor — твой друг

Не пиши expressions вслепую:
1. Открой Expression Editor (кнопка `fx`)
2. Слева видишь доступные данные
3. Кликай на поля → автоподстановка
4. Внизу видишь результат ДО сохранения

### 6. Читай ошибки внимательно

```
ERROR: The property "massage" does not exist
                      ^^^^^^^^
```
Опечатка: `massage` вместо `message`

```
ERROR: Cannot read property 'chat' of undefined
                              ^^^
```
`$json.message` = undefined → проверь что приходит в эту ноду

### 7. Валидируй конфигурацию

Перед запуском используй валидатор n8n (если доступен):
```
n8n-mcp validate_workflow
n8n-mcp validate_node
```

---

## Чек-лист перед каждой нодой

```
□ Понимаю какие данные приходят на вход
□ Понимаю какие данные должны выйти
□ Credential подключен (если нужен)
□ Все required поля заполнены
□ Expressions ссылаются на существующие поля
□ Протестировал с реальными данными
```

---

## Паттерны построения workflows

### Паттерн 1: Telegram → Action → Response

```
[Telegram Trigger]
       ↓
[Typing Action]     ← Сразу показываем что бот думает
       ↓
[Your Logic]
       ↓
[Send Response]
       ↓
[Error Handler]     ← На случай если что-то упадёт
```

**Порядок разработки:**
1. Telegram Trigger → Test (отправь сообщение боту)
2. + Typing Action → Test
3. + Простой Send Response (текст "OK") → Test
4. Теперь добавляй логику посередине

### Паттерн 2: Switch роутинг

```
[Trigger]
    ↓
[Switch]
    ├── Output 0: условие A → [Action A]
    ├── Output 1: условие B → [Action B]
    └── Output 2: fallback  → [Default Action]
```

**Порядок разработки:**
1. Switch с 1 условием → Test
2. Добавляй условия по одному
3. Fallback всегда последний

### Паттерн 3: Файловые операции

```
[Get File from Telegram]
       ↓
[Check file type]       ← Проверка mime-type
       ↓
[Process file]          ← Extract/Convert
       ↓
[Upload to Drive]
       ↓
[Confirm to user]
```

**Порядок разработки:**
1. Get File → Pin data с реальным файлом
2. + Check type (начни с одного типа, например PDF)
3. + Process (сначала без опций)
4. + Upload (сначала hardcode folder ID)
5. Когда работает — добавляй другие типы

---

## Отладка типичных проблем

### Проблема: "Cannot read property X of undefined"

**Диагностика:**
```javascript
// В Set node добавь:
{
  "has_message": {{ $json.message !== undefined }},
  "has_chat": {{ $json.message?.chat !== undefined }},
  "full_json": {{ JSON.stringify($json) }}
}
```

**Частые причины:**
- Данные приходят из другой ноды (нужно `$('NodeName').item.json`)
- После Switch данные меняют структуру
- Ноды выполняются не в том порядке

### Проблема: Expression не работает

**Отладка:**
1. Открой Expression Editor
2. Попробуй `{{ $json }}` — видишь данные?
3. Попробуй `{{ JSON.stringify($json) }}` — что внутри?
4. Попробуй `{{ Object.keys($json) }}` — какие поля есть?

### Проблема: Telegram не отправляет файл

**Причины:**
- File ID истёк (живёт ~1 час)
- Файл слишком большой (>50MB для отправки, >20MB для получения)
- Binary property неправильное имя

**Диагностика:**
```javascript
// Проверь что есть в binary
{{ Object.keys($binary) }}
// Обычно: ["data"] или ["file"]
```

### Проблема: Google API ошибка

**403 Forbidden:**
- Проверь scopes в OAuth consent screen
- Проверь что API включен в Google Cloud

**401 Unauthorized:**
- Переподключи credential (Sign in with Google заново)

**404 Not Found:**
- File/Folder ID неправильный
- Файл удалён или нет доступа

---

## Шаблоны expressions

### Telegram данные

```javascript
// Chat ID
{{ $('TelegramTrigger').item.json.message.chat.id }}

// Username
{{ $('TelegramTrigger').item.json.message.from.username || 'unknown' }}

// Текст сообщения
{{ $('TelegramTrigger').item.json.message.text }}

// File ID документа
{{ $('TelegramTrigger').item.json.message.document.file_id }}

// Имя файла
{{ $('TelegramTrigger').item.json.message.document.file_name }}

// Лучшее фото (максимальное разрешение)
{{ $('TelegramTrigger').item.json.message.photo.slice(-1)[0].file_id }}
```

### Проверки и fallback

```javascript
// С fallback значением
{{ $json.message?.text || 'Нет текста' }}

// Проверка существования
{{ $json.message?.document ? 'Есть документ' : 'Нет документа' }}

// Тип сообщения
{{
  $json.message.text ? 'text' :
  $json.message.document ? 'document' :
  $json.message.photo ? 'photo' :
  $json.message.voice ? 'voice' :
  'unknown'
}}
```

### Форматирование

```javascript
// Дата/время
{{ $now.format('yyyy-MM-dd HH:mm:ss') }}

// Имя файла с датой
{{ 'file_' + $now.format('yyyyMMdd_HHmmss') + '.pdf' }}

// Размер файла в KB
{{ Math.round($json.file_size / 1024) + ' KB' }}
```

---

## Workflow валидация (Claude + n8n MCP)

Перед запуском проси Claude проверить workflow:

```
Провалидируй этот workflow:
[вставить JSON]

Проверь:
1. Все ли connections правильные
2. Все ли required поля заполнены
3. Есть ли expressions с потенциальными ошибками
4. Есть ли обработка ошибок
```

---

## Порядок разработки Шарю-за-Google

### Фаза 1: Минимальный работающий бот

```
День 1:
1. Telegram Trigger → Typing → "Привет!" response
   Тест: отправить любое сообщение

2. + Switch (text vs document)
   Тест: отправить текст, потом файл

3. + Ответ "Получил файл: {filename}"
   Тест: отправить PDF
```

### Фаза 2: Google Drive

```
День 2:
1. Создать credential
   Тест: открыть n8n, проверить что авторизация прошла

2. Google Drive node: List files
   Тест: получить список файлов в папке

3. + Upload file (hardcode folder ID)
   Тест: загрузить тестовый файл
```

### Фаза 3: Соединить

```
День 3:
1. Telegram file → Download → Upload to Drive
   Тест: отправить PDF, проверить на Drive

2. + Подтверждение пользователю
   Тест: получить ссылку на файл
```

---

## Итого: 10 заповедей

1. **Тестируй каждый шаг** — не собирай workflow вслепую
2. **Сначала hardcode** — потом expressions
3. **Pin Data** — сохраняй успешные результаты
4. **Expression Editor** — не пиши выражения в поле напрямую
5. **Читай ошибки** — они говорят что не так
6. **Проверяй входные данные** — добавляй debug Set nodes
7. **Один тип файла** — потом добавляй остальные
8. **Error workflow** — всегда имей обработку ошибок
9. **Документируй** — Sticky Notes в workflow
10. **Инкрементально** — маленькие шаги, частые тесты

---

**Последнее обновление:** 2026-02-01

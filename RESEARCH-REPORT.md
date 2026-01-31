# Исследование: Telegram Bot + Google Sheets + n8n

**Дата:** 2026-02-01
**Проект:** VED-Bot (Manager.VED)

---

## 1. Анализ шаблонов n8n

### Релевантные шаблоны

| ID | Название | Views | Ключевые особенности |
|----|----------|-------|---------------------|
| [3798](https://n8n.io/workflows/3798) | Session-Based Telegram Chatbot + Google Sheets | 40K+ | Сессии в Sheets, команды /new /resume /summary |
| [4696](https://n8n.io/workflows/4696) | Conversational Telegram Bot GPT-4o | 82K+ | Текст + голос, Typing Action |
| [5291](https://n8n.io/workflows/5291) | Task Manager + Telegram + Sheets + GPT-4o | 20K+ | **Google Sheets Tool для AI Agent** |
| [4110](https://n8n.io/workflows/4110) | Clone Viral TikToks | 84K+ | TG Trigger → Sheets → AI → Multi-platform |

### Ключевой шаблон: #5291 (Task Manager)

Архитектура идеально подходит для VED-Bot:

```
┌─────────────────┐
│ Telegram Trigger│
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌──────────────────┐
│    AI Agent     │◄────│ Google Sheets    │
│   (GPT-4o)      │     │ Tool (Read)      │
└────────┬────────┘     └──────────────────┘
         │              ┌──────────────────┐
         │◄─────────────│ Google Sheets    │
         │              │ Tool (Append)    │
         ▼              └──────────────────┘
┌─────────────────┐     ┌──────────────────┐
│ Send Response   │     │ Google Sheets    │
│   (Telegram)    │◄────│ Tool (Update)    │
└─────────────────┘     └──────────────────┘
```

**Преимущества:**
- AI сам решает когда читать/писать в таблицу
- Не нужны отдельные ветки для Sheets операций
- Естественный диалог: "Запиши в таблицу..." / "Найди в справочнике..."

---

## 2. Best Practices: Google Sheets API

### Операции

| Метод | Использование | Rate Limit |
|-------|--------------|------------|
| `values.append` | Добавить строку в конец | 60 req/min |
| `values.update` | Обновить конкретные ячейки | 60 req/min |
| `values.batchUpdate` | Множественные операции за 1 запрос | 60 req/min |
| `values.get` | Чтение данных | 60 req/min |

### Рекомендации

1. **valueInputOption**:
   - `USER_ENTERED` — значения парсятся (даты, числа)
   - `RAW` — значения как есть (текст)

2. **Структура таблицы для VED-Bot**:
   ```
   Лист "requests":
   | timestamp | user_id | username | message | response | status | session_id |

   Лист "products":
   | name | category | hs_code_hint | description_template | materials |

   Лист "exports":
   | timestamp | user_id | product_name | full_description | files |
   ```

3. **Оптимизация**:
   - Использовать `batchUpdate` для множественных записей
   - Кэшировать справочники в памяти агента
   - Лимитировать чтение: `range: "A1:F100"`

**Источники:**
- [Append Method](https://developers.google.com/workspace/sheets/api/reference/rest/v4/spreadsheets.values/append)
- [Basic Writing](https://developers.google.com/workspace/sheets/api/samples/writing)
- [Batch Update](https://developers.google.com/sheets/api/guides/batchupdate)

---

## 3. Best Practices: Telegram Bot API

### Webhook vs Polling

| Аспект | Webhook | Polling |
|--------|---------|---------|
| Latency | Низкая (мгновенно) | Высокая (интервал опроса) |
| Scalability | Высокая | Ограничена |
| Setup | Требует HTTPS, домен | Проще настройка |
| **Рекомендация** | **Production** | Development/тесты |

### Rate Limits

| Контекст | Лимит |
|----------|-------|
| Личные чаты | 1 msg/sec на пользователя |
| Bulk (разным пользователям) | 30 msg/sec общий |
| Группы | 20 msg/min на группу |

### Session Management

```javascript
// Рекомендуемая структура сессии
{
  session_id: "uuid",
  user_id: 530405361,
  created_at: "2026-02-01T10:00:00Z",
  expires_at: "2026-02-01T11:00:00Z",
  status: "active", // active | completed | expired
  documents: [],
  context: {}
}
```

### Команды для VED-Bot

| Команда | Действие |
|---------|----------|
| `/start` | Приветствие, справка |
| `/new` | Новая сессия (описание товара) |
| `/done` | Завершить сессию, экспорт |
| `/cancel` | Отменить текущую сессию |
| `/status` | Показать статус сессии |
| `/export` | Экспорт в Google Sheets |

**Источники:**
- [Telegram Bot API](https://core.telegram.org/bots/api)
- [Developer's Guide 2025](https://stellaray777.medium.com/a-developers-guide-to-building-telegram-bots-in-2025-dbc34cd22337)
- [Webhook Best Practices](https://wnexus.io/the-complete-guide-to-telegram-bot-development-in-2025/)

---

## 4. Рекомендуемая архитектура для VED-Bot

### Вариант A: Google Sheets Tool для AI Agent (рекомендуется)

```
┌─────────────────────────────────────────────────────────────┐
│                     VED-Bot-Pro v4                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐                                           │
│  │ TelegramTrig │──►┌──────────────┐                        │
│  └──────────────┘   │ TypingAction │                        │
│                     └──────┬───────┘                        │
│                            │                                │
│                     ┌──────▼───────┐                        │
│                     │    Switch    │                        │
│                     │ (text/voice) │                        │
│                     └──────┬───────┘                        │
│                            │                                │
│  ┌─────────────────────────▼─────────────────────────────┐  │
│  │                    AI Agent                           │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │ Tools:                                          │  │  │
│  │  │  • Google Sheets Tool (Read справочники)        │  │  │
│  │  │  • Google Sheets Tool (Append заявки)           │  │  │
│  │  │  • Google Sheets Tool (Update статусы)          │  │  │
│  │  │  • HTTP Tool (внешние API)                      │  │  │
│  │  │  • Wikipedia Tool                               │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │ Memory: Postgres (Supabase)                     │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────┬───────────────────────────────┘  │
│                          │                                  │
│                   ┌──────▼───────┐                          │
│                   │ SendResponse │                          │
│                   └──────────────┘                          │
└─────────────────────────────────────────────────────────────┘
```

**Преимущества:**
- Агент сам решает когда обращаться к таблице
- Меньше нод, проще поддержка
- Естественное взаимодействие

### Вариант B: Явные ноды Google Sheets

```
Telegram → Switch → [text/voice] → AI Agent → SendResponse
                                       │
                                       ├──► LogToSheets (append)
                                       └──► ReadFromSheets (lookup)
```

**Преимущества:**
- Полный контроль над операциями
- Легче дебажить

---

## 5. План интеграции для VED-Bot

### Этап 1: Настройка Google Sheets (готово частично)

- [x] Создан workflow VED-Bot-GoogleSheets.json
- [ ] Создать Google OAuth2 credential в n8n
- [ ] Создать таблицу с листами

### Этап 2: Добавить Google Sheets Tools в AI Agent

```json
{
  "name": "GoogleSheetsReadProducts",
  "type": "n8n-nodes-base.googleSheetsTool",
  "parameters": {
    "operation": "read",
    "sheetName": "products",
    "toolDescription": "Поиск информации о товаре в справочнике. Используй для поиска шаблонов описаний, кодов ТН ВЭД, материалов."
  }
}
```

### Этап 3: Логирование

Добавить автоматическую запись после каждого ответа:
```
AI Agent → SendResponse → LogToSheets
```

### Этап 4: Экспорт

Команда `/export` → собрать данные сессии → записать в лист "exports"

---

## 6. Чеклист внедрения

- [ ] Создать Google Cloud Project
- [ ] Включить Google Sheets API
- [ ] Создать OAuth2 credentials (redirect URI: `https://marydgd.app.n8n.cloud/rest/oauth2-credential/callback`)
- [ ] Создать таблицу с 3 листами (requests, products, exports)
- [ ] Добавить credential в n8n
- [ ] Добавить Google Sheets Tools в VED_AI_Agent
- [ ] Обновить системный промпт агента (добавить инструкции по работе с таблицей)
- [ ] Протестировать чтение/запись
- [ ] Активировать в production

---

## 7. Полезные ссылки

### Документация
- [n8n Google Sheets Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/)
- [Google Sheets API v4](https://developers.google.com/sheets/api)
- [Telegram Bot API](https://core.telegram.org/bots/api)

### Шаблоны n8n
- [Session-based Chatbot](https://n8n.io/workflows/3798)
- [Task Manager + Sheets](https://n8n.io/workflows/5291)
- [Conversational Bot GPT-4o](https://n8n.io/workflows/4696)

### Статьи
- [Telegram Bot Development 2025](https://wnexus.io/the-complete-guide-to-telegram-bot-development-in-2025/)
- [Google Sheets + n8n Integration](https://n8n.io/integrations/google-sheets/and/telegram/)

---

**Подготовлено:** Claude Code
**Последнее обновление:** 2026-02-01

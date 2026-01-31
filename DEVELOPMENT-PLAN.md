# План разработки: Шарю за Google

**Цель:** Telegram бот с полным доступом к Google Drive и Google Sheets

---

## Архитектура

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         TELEGRAM BOT                                    │
│  Принимает: текст, голос, фото, документы (PDF, DOCX, XLSX, etc.)      │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           AI AGENT                                       │
│  GPT-4o с инструментами:                                                │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────────────────┐│
│  │ Google Drive    │ │ Google Sheets   │ │ Document Analyzer           ││
│  │ Tool            │ │ Tool            │ │ (PDF/DOCX/XLSX reader)      ││
│  └─────────────────┘ └─────────────────┘ └─────────────────────────────┘│
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────────────────┐│
│  │ File Compare    │ │ OCR Tool        │ │ Search Tool                 ││
│  │ Tool            │ │ (GPT-4o Vision) │ │ (find in Drive)             ││
│  └─────────────────┘ └─────────────────┘ └─────────────────────────────┘│
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
┌─────────────────────┐ ┌───────────────┐ ┌─────────────────────┐
│   Google Drive      │ │ Google Sheets │ │   Telegram          │
│   - Upload          │ │ - Read/Write  │ │   - Send response   │
│   - Download        │ │ - Search      │ │   - Send files      │
│   - List/Search     │ │ - Update      │ │                     │
│   - Share           │ │               │ │                     │
└─────────────────────┘ └───────────────┘ └─────────────────────┘
```

---

## Этапы разработки

### Этап 1: Базовый бот с приёмом файлов

**Цель:** Бот принимает любые документы и сохраняет на Google Drive

**Компоненты:**
```
Telegram Trigger
    ↓
Switch (text/voice/photo/document)
    ↓
[document branch]
    ↓
Get File (Telegram)
    ↓
Upload to Google Drive
    ↓
Send confirmation (Telegram)
```

**Поддерживаемые форматы:**
| Тип | Расширения |
|-----|------------|
| Документы | PDF, DOCX, DOC, ODT, RTF |
| Таблицы | XLSX, XLS, CSV, ODS |
| Изображения | JPG, PNG, GIF, WebP |
| Прочее | TXT, JSON, XML, ZIP |

**Задачи:**
- [ ] Telegram Trigger для всех типов сообщений
- [ ] Switch роутинг по типу контента
- [ ] Telegram node для скачивания файлов
- [ ] Google Drive node для загрузки
- [ ] Структура папок на Drive (по пользователям/датам)

---

### Этап 2: Чтение и анализ документов

**Цель:** Бот читает содержимое файлов и отвечает на вопросы

**Компоненты:**
```
Document received
    ↓
Extract from File
    ├── PDF → Extract From PDF
    ├── XLSX → Extract From XLSX
    ├── CSV → Extract From CSV
    ├── DOCX → Binary Input Loader (LangChain)
    └── Image → GPT-4o Vision OCR
    ↓
AI Agent (analyze content)
    ↓
Send response
```

**Ноды n8n:**
| Формат | Нода |
|--------|------|
| PDF | `extractFromFile` (operation: pdf) |
| XLSX/XLS | `extractFromFile` (operation: xlsx/xls) |
| CSV | `extractFromFile` (operation: csv) |
| DOCX | `documentBinaryInputLoader` (loader: docxLoader) |
| Изображения | `openAi` (GPT-4o Vision) |
| TXT | `extractFromFile` (operation: text) |

**Задачи:**
- [ ] Определение типа файла по mime-type
- [ ] Роутинг на соответствующий экстрактор
- [ ] Передача контента в AI Agent
- [ ] Промпт для анализа документов

---

### Этап 3: Интеграция с Google Drive

**Цель:** Бот видит все файлы на Drive, может искать и скачивать

**Операции Google Drive:**
| Операция | Описание |
|----------|----------|
| `list` | Список файлов в папке |
| `search` | Поиск по имени/содержимому |
| `download` | Скачать файл |
| `upload` | Загрузить файл |
| `copy` | Копировать файл |
| `move` | Переместить файл |
| `share` | Поделиться файлом |
| `delete` | Удалить файл |

**Структура папок:**
```
Google Drive/
├── Шарю-за-Google/
│   ├── Входящие/
│   │   ├── 2026-02/
│   │   │   ├── user_123456/
│   │   │   │   ├── document1.pdf
│   │   │   │   └── invoice.xlsx
│   │   └── ...
│   ├── Обработанные/
│   └── Архив/
```

**Задачи:**
- [ ] Google Drive OAuth2 credential
- [ ] Создание структуры папок
- [ ] Команда `/list` — показать файлы
- [ ] Команда `/search <query>` — поиск
- [ ] Команда `/get <filename>` — скачать

---

### Этап 4: Интеграция с Google Sheets

**Цель:** Бот видит все таблицы, может читать/писать

**Операции Google Sheets:**
| Операция | Описание |
|----------|----------|
| `read` | Чтение данных |
| `append` | Добавить строку |
| `update` | Обновить строку |
| `clear` | Очистить |
| `getMany` | Список всех таблиц |

**Таблицы бота:**
| Таблица | Назначение |
|---------|------------|
| `logs` | Лог всех операций |
| `files` | Индекс загруженных файлов |
| `users` | Данные пользователей |
| `comparisons` | История сравнений |

**Задачи:**
- [ ] Google Sheets OAuth2 credential
- [ ] Логирование всех операций
- [ ] Команда `/sheets` — список таблиц
- [ ] Команда `/sheet <name>` — открыть таблицу
- [ ] Инлайн-кнопки для навигации

---

### Этап 5: Сравнение документов

**Цель:** Сравнить присланный документ с файлами на Drive

**Логика:**
```
User sends document
    ↓
Extract content from new document
    ↓
Search similar files on Drive
    ↓
Download and extract content from Drive files
    ↓
AI Agent compares documents
    ↓
Generate comparison report
    ↓
Save report to Drive + Send to user
```

**Типы сравнения:**
1. **Структурное** — колонки, строки, формат
2. **Содержательное** — текст, числа, даты
3. **Semantic** — смысловое сравнение через AI

**Задачи:**
- [ ] Поиск похожих файлов по имени
- [ ] Извлечение контента из обоих файлов
- [ ] Промпт для сравнения
- [ ] Генерация отчёта (Markdown/PDF)
- [ ] Сохранение отчёта на Drive

---

### Этап 6: AI Agent с инструментами

**Цель:** Агент сам решает какие инструменты использовать

**Tools для AI Agent:**

```javascript
// Google Drive Tool
{
  name: "google_drive",
  description: "Работа с Google Drive: загрузка, скачивание, поиск файлов",
  operations: ["upload", "download", "search", "list", "share"]
}

// Google Sheets Tool
{
  name: "google_sheets",
  description: "Работа с Google Sheets: чтение, запись, поиск таблиц",
  operations: ["read", "append", "update", "list_sheets"]
}

// Document Analyzer Tool
{
  name: "analyze_document",
  description: "Анализ содержимого документа",
  formats: ["pdf", "docx", "xlsx", "csv", "txt", "image"]
}

// Compare Tool
{
  name: "compare_documents",
  description: "Сравнение двух документов",
  output: "difference_report"
}
```

**Системный промпт:**
```
Ты AI-ассистент с полным доступом к Google Drive и Google Sheets пользователя.

Твои возможности:
1. Принимать любые документы и сохранять на Drive
2. Читать содержимое PDF, DOCX, XLSX, изображений
3. Искать файлы на Drive по имени и содержимому
4. Читать и редактировать таблицы Google Sheets
5. Сравнивать документы между собой
6. Генерировать отчёты и сохранять на Drive

Правила:
- Всегда подтверждай операции с файлами
- При загрузке файла — сохраняй в папку Входящие/{дата}
- При сравнении — создавай отчёт в папке Отчёты
- Логируй все операции в таблицу logs
```

---

## Команды бота

| Команда | Описание |
|---------|----------|
| `/start` | Приветствие, справка |
| `/help` | Список команд |
| `/list [папка]` | Список файлов |
| `/search <запрос>` | Поиск на Drive |
| `/get <файл>` | Скачать файл |
| `/sheets` | Список таблиц |
| `/compare` | Сравнить последний файл с Drive |
| `/logs` | Показать лог операций |
| `/settings` | Настройки |

---

## Технический стек

| Компонент | Технология |
|-----------|------------|
| Автоматизация | n8n Cloud |
| AI | OpenAI GPT-4o |
| Мессенджер | Telegram Bot API |
| Хранилище | Google Drive |
| Таблицы | Google Sheets |
| OCR | GPT-4o Vision |
| Память | PostgreSQL (Supabase) |

---

## Credentials

| Сервис | Тип |
|--------|-----|
| Telegram | Bot Token |
| OpenAI | API Key |
| Google Drive | OAuth2 |
| Google Sheets | OAuth2 (тот же) |
| PostgreSQL | Connection String |

---

## Файловая структура проекта

```
sharu-za-google/
├── README.md
├── DEVELOPMENT-PLAN.md          # Этот файл
├── RESEARCH-REPORT.md           # Исследование
├── SETUP.md                     # Инструкции по настройке
├── workflows/
│   ├── main-bot.json            # Основной workflow
│   ├── file-processor.json      # Обработка файлов
│   ├── drive-manager.json       # Работа с Drive
│   ├── sheets-manager.json      # Работа с Sheets
│   └── document-compare.json    # Сравнение документов
└── prompts/
    ├── system-prompt.md         # Системный промпт агента
    └── compare-prompt.md        # Промпт для сравнения
```

---

## Timeline

| Этап | Описание | Статус |
|------|----------|--------|
| 1 | Базовый бот + загрузка на Drive | 🔲 |
| 2 | Чтение документов | 🔲 |
| 3 | Google Drive интеграция | 🔲 |
| 4 | Google Sheets интеграция | 🔲 |
| 5 | Сравнение документов | 🔲 |
| 6 | AI Agent с tools | 🔲 |

---

## Следующий шаг

**Начать с Этапа 1:**
1. Создать Telegram бота через @BotFather
2. Настроить Google OAuth2
3. Создать базовый workflow для приёма файлов

---

**Подготовлено:** Claude Code
**Дата:** 2026-02-01

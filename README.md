# Шарю за Google

Интеграция Telegram бота с Google Sheets через n8n.

## Что это

Workflow для n8n, который позволяет:
- Логировать запросы пользователей в Google Sheets
- Читать данные из справочников
- Экспортировать результаты в таблицу

## Структура

```
sharu-za-google/
├── README.md
├── RESEARCH-REPORT.md      # Исследование best practices
└── workflows/
    └── VED-Bot-GoogleSheets.json
```

## Настройка

### 1. Google Cloud Console

1. Создай проект в [Google Cloud Console](https://console.cloud.google.com/)
2. Включи Google Sheets API
3. Создай OAuth2 credentials (Web application)
4. Redirect URI: `https://YOUR-N8N-URL/rest/oauth2-credential/callback`

### 2. n8n

1. Создай credential: Google Sheets OAuth2 API
2. Импортируй workflow из `workflows/`
3. Активируй

### 3. Google Sheets

Создай таблицу с листами:
- `requests` — логирование
- `products` — справочник
- `exports` — результаты

## Технологии

- n8n
- Google Sheets API v4
- Telegram Bot API

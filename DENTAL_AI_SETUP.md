# 🦷 Инструкция по сборке AI-Receptionist (Dental-Lux Case)

Этот файл содержит полный финальный код и настройки для создания голосового робота-администратора для стоматологии.

---

## 🏗 Архитектура
1.  **ElevenLabs (AI Agent)**: Принимает звонок, говорит голосом, собирает данные.
2.  **n8n (Webhook + Logic)**: Получает данные, обрабатывает дату, создает сущности в CRM.
3.  **AmoCRM**: Хранит клиентов, сделки и календарь записей.
4.  **Telegram**: Уведомляет врача/админа о новой записи.

---

## 1. Настройка ElevenLabs (Prompt)
**Role**: `Dentist Receptionist`
**Model**: `Turbo v2`
**Voice**: `Kate` (или `Rachel`)

### Системный Промпт:
```text
=== ГЛАВНЫЙ АЛГОРИТМ ===
1. Приветствие: "Добрый день, стоматология Denta-Lux, меня зовут Анна. Чем могу помочь?"
2. Выясни жалобу: "Что вас беспокоит? Боль острая?"
3. Узнай статус: "Вы у нас уже были раньше?"
4. **ПРОВЕРКА ВРЕМЕНИ**:
   - Если клиент предлагает время (например, "Завтра в 14:00"), СНАЧАЛА вызови инструмент `check_availability`.
   - Если инструмент вернул "busy" -> Скажи: "К сожалению, на 14:00 уже занято. Есть свободное окошко в [alternative_time]. Записать?"
   - Если инструмент вернул "available" -> Скажи: "Да, это время свободно." и переходи к п.5.
5. Если клиент согласен — спроси Имя и Телефон.
6. ФИНАЛ: Четко проговори: "Записала вас на [дата/время]. Адрес: Ленина, 15. Ждем вас!".
7. Только после подтверждения вызови инструмент save_dental_lead.

=== ПРАВИЛА ===
✗ Не называй адрес в начале, только в конце.
✗ Если спрашивают "Дорого?" -> "Мы используем материалы 5-го поколения, это надежно. Есть рассрочка."

=== ИНСТРУМЕНТЫ (Tools) ===

1. `check_availability (date, time)`
   - Описание: Проверяет, свободен ли слот.
   - Используй ВСЕГДА, когда клиент называет конкретное время.

2. `save_dental_lead (patient_name, phone_number, ...)`
   - Описание: Финальная запись в CRM.
   - Данные:
     - patient_name (Имя)
     - complaint (Жалоба кратко)
     - phone_number (Телефон)
     - appointment_time (Формат YYYY-MM-DD HH:MM, например 2026-01-28 14:00)
```

---

## 2. Настройка AmoCRM и Google Calendar
### Синхронизация:
Чтобы бот видел занятые слоты, которые создал человек-администратор:
1.  Зайдите в AmoCRM -> Настройки -> Интеграции.
2.  Включите **Google Calendar**.
3.  Теперь любая задача/сделка в Amo будет занимать слот в календаре.

### Воронка (Этапы):
1.  **НОВАЯ ЗАЯВКА** (Сюда падают лиды от бота)
2.  **ЗАПИСАН НА ПРИЁМ** (Сюда переносит админ после проверки)
3.  **ПОДТВЕРЖДЕНО** (Звонок за 24 часа)
4.  **ПРИЕХАЛ В КЛИНИКУ**
5.  **УСПЕШНО РЕАЛИЗОВАНО**

### Поля Сделки (Custom Fields):
*   `Жалоба` (Text)
*   `Дата записи` (Text или DateTime) — **Снять галочку "Только из API"!**

---

## 3. Код n8n (Workflow JSON)
Скопируйте этот код и импортируйте в n8n (Top Right Menu -> Import from File/JSON).

```json
{
  "nodes": [
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "dental-lead",
        "options": {}
      },
      "id": "51570f29-73c8-4ac1-a09b-03c84dd40f61",
      "name": "ElevenLabs Webhook",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [
        64,
        -16
      ],
      "webhookId": "dental-lead"
    },
    {
      "parameters": {
        "assignments": {
          "assignments": [
            {
              "id": "patient_name",
              "name": "patient_name",
              "value": "={{ $json.body.patient_name ?? $json.query.patient_name }}",
              "type": "string"
            },
            {
              "id": "phone_number",
              "name": "phone_number",
              "value": "={{ $json.body.phone_number ?? $json.query.phone_number }}",
              "type": "string"
            },
            {
              "id": "appointment_time",
              "name": "appointment_time",
              "value": "={{ $json.body.appointment_time ?? $json.query.appointment_time }}",
              "type": "string"
            },
            {
              "id": "complaint",
              "name": "complaint",
              "value": "={{ $json.body.complaint ?? $json.query.complaint }}",
              "type": "string"
            }
          ]
        },
        "options": {}
      },
      "id": "2902585b-004e-4e07-b187-c86d3f06687a",
      "name": "Extract Fields",
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [
        256,
        -16
      ]
    },
    {
      "parameters": {
        "chatId": "669820318",
        "text": "=🦷 *Новая заявка с AI-бота*\n\n👤 *Пациент:* {{ $('Extract Fields').item.json.patient_name }}\n📱 *Телефон:* {{ $('Extract Fields').item.json.phone_number }}\n🕐 *Время:* {{ $('Extract Fields').item.json.appointment_time }}\n💬 *Жалоба:* {{ $('Extract Fields').item.json.complaint }}",
        "additionalFields": {
          "parse_mode": "Markdown"
        }
      },
      "id": "38263847-358e-4278-a71f-25d33e931078",
      "name": "Telegram: Уведомление",
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1.2,
      "position": [
        1088,
        -16
      ],
      "webhookId": "6a4af7de-198d-4195-9350-869315f8bcad",
      "credentials": {
        "telegramApi": {
          "id": "CFp8Q00qzKWJdJ62",
          "name": "Стоматолгия"
        }
      }
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://ruslan1993.amocrm.ru/api/v4/contacts",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "=[{\"name\": \"{{ $json.patient_name }}\", \"custom_fields_values\": [{\"field_code\": \"PHONE\", \"values\": [{\"value\": \"{{ $json.phone_number }}\"}]}]}]",
        "options": {
          "response": {
            "response": {
              "responseFormat": "json"
            }
          }
        }
      },
      "id": "amocrm-contact",
      "name": "AmoCRM Создать контакт",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        592,
        -16
      ],
      "credentials": {
        "httpHeaderAuth": {
          "id": "peuRYKcQ7BGm9pPJ",
          "name": "Амо СРМ Стоматология"
        }
      }
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://ruslan1993.amocrm.ru/api/v4/leads",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "=[{\"name\": \"{{ $('Парсинг даты').item.json.deal_name }}\", \"pipeline_id\": 10525470, \"custom_fields_values\": [{\"field_id\": 1127180, \"values\": [{\"value\": \"{{ $('Парсинг даты').item.json.appointment_time }}\"}]}], \"_embedded\": {\"contacts\": [{\"id\": {{ $json._embedded.contacts[0].id }}}]}}]",
        "options": {
          "response": {
            "response": {
              "responseFormat": "json"
            }
          }
        }
      },
      "id": "amocrm-lead",
      "name": "AmoCRM Создать сделку",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        736,
        -16
      ],
      "credentials": {
        "httpHeaderAuth": {
          "id": "peuRYKcQ7BGm9pPJ",
          "name": "Амо СРМ Стоматология"
        }
      }
    },
    {
      "parameters": {
        "jsCode": "const data = $input.first().json;\nconst appointmentText = data.appointment_time || '';\n\n// Часовой пояс Екатеринбурга (UTC+5)\nconst TIMEZONE_OFFSET_HOURS = 5;\n\nfunction parseDate(text) {\n  // ISO формат (2026-01-28 13:00)\n  const isoMatch = text.match(/(\\d{4})-(\\d{2})-(\\d{2})[T\\s](\\d{1,2}):(\\d{2})/);\n  if (isoMatch) {\n    const [_, year, month, day, hours, minutes] = isoMatch;\n    const localDate = new Date(Date.UTC(\n      parseInt(year), \n      parseInt(month) - 1, \n      parseInt(day), \n      parseInt(hours) - TIMEZONE_OFFSET_HOURS, \n      parseInt(minutes)\n    ));\n    return localDate;\n  }\n  \n  // Русский текст\n  const now = new Date();\n  const localNow = new Date(now.getTime() + TIMEZONE_OFFSET_HOURS * 60 * 60 * 1000);\n  let targetDate = new Date(localNow);\n  const lowerText = text.toLowerCase();\n  \n  if (lowerText.includes('завтра')) {\n    targetDate.setUTCDate(targetDate.getUTCDate() + 1);\n  } else if (lowerText.includes('послезавтра')) {\n    targetDate.setUTCDate(targetDate.getUTCDate() + 2);\n  }\n  \n  const timeMatch = text.match(/(\\d{1,2})[:\\s-]?(\\d{2})?/);\n  if (timeMatch) {\n    const hours = parseInt(timeMatch[1]);\n    const minutes = timeMatch[2] ? parseInt(timeMatch[2]) : 0;\n    targetDate.setUTCHours(hours - TIMEZONE_OFFSET_HOURS, minutes, 0, 0);\n  } else {\n    if (lowerText.includes('утр')) targetDate.setUTCHours(9 - TIMEZONE_OFFSET_HOURS, 0, 0, 0);\n    else if (lowerText.includes('вечер')) targetDate.setUTCHours(18 - TIMEZONE_OFFSET_HOURS, 0, 0, 0);\n    else targetDate.setUTCHours(12 - TIMEZONE_OFFSET_HOURS, 0, 0, 0);\n  }\n  \n  return targetDate;\n}\n\nconst appointmentDate = parseDate(appointmentText);\nconst appointmentTimestamp = Math.floor(appointmentDate.getTime() / 1000);\nconst reminderTimestamp = appointmentTimestamp - 3600;\n\nconst displayDate = new Date(appointmentDate.getTime() + TIMEZONE_OFFSET_HOURS * 60 * 60 * 1000);\nconst timeFormatted = displayDate.getUTCHours().toString().padStart(2, '0') + ':' + displayDate.getUTCMinutes().toString().padStart(2, '0');\n\n// Новый формат: Пациент — причина\nconst deal_name = `${data.patient_name} \\u2014 ${data.complaint}`;\n\nreturn [{\n  json: {\n    ...data,\n    appointment_timestamp: appointmentTimestamp,\n    reminder_timestamp: reminderTimestamp,\n    time_formatted: timeFormatted,\n    deal_name: deal_name\n  }\n}];"
      },
      "id": "parse-datetime",
      "name": "Парсинг даты",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        448,
        -16
      ]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://ruslan1993.amocrm.ru/api/v4/tasks",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "=[{\"entity_id\": {{ $('AmoCRM Создать сделку').item.json._embedded.leads[0].id }}, \"entity_type\": \"leads\", \"text\": \"Приём: {{ $('Парсинг даты').item.json.patient_name }}\\nЖалоба: {{ $('Парсинг даты').item.json.complaint }}\\nТел: {{ $('Парсинг даты').item.json.phone_number }}\", \"complete_till\": {{ $('Парсинг даты').item.json.reminder_timestamp }}, \"task_type_id\": 1}]",
        "options": {
          "response": {
            "response": {
              "responseFormat": "json"
            }
          }
        }
      },
      "id": "amocrm-task",
      "name": "AmoCRM Создать задачу",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        912,
        -16
      ],
      "credentials": {
        "httpHeaderAuth": {
          "id": "peuRYKcQ7BGm9pPJ",
          "name": "Амо СРМ Стоматология"
        }
      }
    }
  ],
  "connections": {
    "ElevenLabs Webhook": {
      "main": [
        [
          {
            "node": "Extract Fields",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Extract Fields": {
      "main": [
        [
          {
            "node": "Парсинг даты",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "AmoCRM Создать контакт": {
      "main": [
        [
          {
            "node": "AmoCRM Создать сделку",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "AmoCRM Создать сделку": {
      "main": [
        [
          {
            "node": "AmoCRM Создать задачу",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Парсинг даты": {
      "main": [
        [
          {
            "node": "AmoCRM Создать контакт",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "AmoCRM Создать задачу": {
      "main": [
        [
          {
            "node": "Telegram: Уведомление",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "pinData": {},
  "meta": {
    "templateCredsSetupCompleted": true,
    "instanceId": "063c8cba689329b722d1b1e1bfe8a02783ad3f5ae1e8b014d1ebde538179b085"
  }
}
```

---

## 4. Настройка проверки времени (n8n)
Чтобы бот мог проверять слоты, создайте **второй** workflow в n8n для инструмента `check_availability`.

### Логика (Имитация):
1.  **Webhook**: Слушает запросы от ElevenLabs.
2.  **Code Node**:
    - Если время `14:00` -> Возвращает `status: busy`, советует `16:30`.
    - Любое другое время -> Возвращает `status: available`.
3.  **Respond to Webhook**: Отправляет JSON-ответ обратно в бота.

### Код Workflow (check_availability):
Файл с кодом создан рядом: `check_availability_workflow.json`. Импортируйте его так же, как основной.

---

## 5. Следующие шаги (масштабирование)
*   Подключить Google Calendar для проверки занятости через n8n.
*   Добавить WhatsApp для подтверждения записей.
*   Использовать VAPI для In-Call latency.

*Created by Antigravity AI.*

# AI-редактирование изображений в QRMe

## Статус: MVP работает на iOS (ветка `feature/ai-image-editor`)

---

## 1. Что сделано (текущая реализация)

### Новые файлы

| Файл | Назначение |
|------|------------|
| `QRme/UI/MediaEditor/QRmeAIEditService.swift` | Сервис AI-редактирования — отправляет фото + промпт в OpenRouter, получает отредактированное фото |
| `QRme/services/Netwoking/MoyaApiService/v3/API/AIEditAPI_V3.swift` | Moya endpoint `POST /v3/ai/edit-image` (заготовка для бэкенда, пока не используется) |

### Изменённые файлы

| Файл | Что изменено |
|------|-------------|
| `QRme/UI/MediaEditor/QRmePhotoEditorViewController.swift` | Добавлена кнопка «✦ Изменить с ИИ ⬡5», prompt overlay, вызов `QRmeAIEditService`, animated stage labels |
| `QRme/QRme-Info.plist` | ATS exception для `151.247.197.68` (upload сервер, можно убрать) |
| `QRme.xcodeproj/project.pbxproj` | Добавлены новые файлы в проект |

### UI в редакторе

В панели "Фон" (background panel) добавлена кнопка **«✦ Изменить с ИИ ⬡5»** — фиолетовая (#7D4FFF), с gold SVG-монеткой (⬡) и ценой "5".

При нажатии открывается fullscreen overlay:
- Заголовок **«Что изменить?»**
- Badge **«⬡ 5 монет»** (gold)
- Текстовое поле с placeholder «Например: сделай фон закатом на пляже»
- Кнопка **«Изменить»** (фиолетовая)
- Кнопка **«Отмена»**
- Return на клавиатуре = отправить

Во время обработки показываются animated labels (crossDissolve transition):
1. «Подготавливаем фото...»
2. «Объясняем нейронке куда смотреть...»
3. «Рисуем детали...»
4. «Почти готово...»

---

## 2. Как пайплайн работает СЕЙЧАС (MVP, OpenRouter)

```
┌─────────────────────────────────────────────────────┐
│ Пользователь в редакторе изображений                │
│ Нажимает «✦ Изменить с ИИ»                          │
│ Вводит: «Добавь на фон медведя на велосипеде»      │
└──────────────────┬──────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────┐
│ Step 1: Подготовка фото (iOS, на устройстве)        │
│                                                      │
│ • Resize до max 1024px по большей стороне            │
│   (UIGraphicsImageRenderer)                          │
│ • JPEG compression quality 0.75                      │
│ • Base64 encode → строка ~100-200KB                  │
└──────────────────┬──────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────┐
│ Step 2: Отправка в OpenRouter (ВРЕМЕННОЕ РЕШЕНИЕ)    │
│                                                      │
│ POST https://openrouter.ai/api/v1/chat/completions  │
│ Headers:                                             │
│   Authorization: Bearer sk-or-v1-...                 │
│   Content-Type: application/json                     │
│                                                      │
│ Body:                                                │
│ {                                                    │
│   "model": "google/gemini-2.5-flash-image",          │
│   "messages": [{                                     │
│     "role": "user",                                  │
│     "content": [                                     │
│       {                                              │
│         "type": "image_url",                         │
│         "image_url": {                               │
│           "url": "data:image/jpeg;base64,/9j/4A..." │
│         }                                            │
│       },                                             │
│       {                                              │
│         "type": "text",                              │
│         "text": "Edit this image: добавь медведя..." │
│       }                                              │
│     ]                                                │
│   }]                                                 │
│ }                                                    │
│                                                      │
│ Timeout: 120 секунд                                  │
│ Стоимость: ~$0.04 за запрос                          │
└──────────────────┬──────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────┐
│ Step 3: Ответ от OpenRouter                         │
│                                                      │
│ {                                                    │
│   "choices": [{                                      │
│     "message": {                                     │
│       "role": "assistant",                           │
│       "content": null,                               │
│       "images": [{                                   │
│         "type": "image_url",                         │
│         "image_url": {                               │
│           "url": "data:image/png;base64,iVBORw0K..." │
│         }                                            │
│       }]                                             │
│     }                                                │
│   }]                                                 │
│ }                                                    │
│                                                      │
│ Изображение приходит как base64 PNG в поле images    │
└──────────────────┬──────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────┐
│ Step 4: Отображение результата (iOS)                │
│                                                      │
│ • Decode base64 → Data → UIImage                     │
│ • workingImage = result                              │
│ • applyCurrentEdits() → обновить imageView           │
│ • Undo state сохранён перед правкой                  │
└─────────────────────────────────────────────────────┘
```

### Проблемы текущего решения

1. **API ключ OpenRouter захардкожен в iOS** — небезопасно, любой может декомпилировать и украсть ключ
2. **Нет контроля расходов** — iOS шлёт запросы напрямую, бэкенд не знает о тратах
3. **Нет списания монет** — стоимость «5 монет» показывается но не списывается
4. **Нет rate limiting** — пользователь может спамить запросами
5. **OpenRouter — лишний посредник** — aggai.ru уже есть, нужно добавить multimodal support

---

## 3. Как пайплайн ДОЛЖЕН работать (через бэкенд QRMe + aggai)

```
┌─────────────────────────────────────────────────────┐
│ iOS: Пользователь вводит промпт                     │
│ • Resize фото до 1024px                             │
│ • JPEG quality 0.75                                  │
│ • Base64 encode                                      │
└──────────────────┬──────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────┐
│ iOS → Backend: POST /v3/ai/edit-image               │
│                                                      │
│ Headers: Authorization: Bearer {JWT пользователя}    │
│ Body: {                                              │
│   "imageBase64": "/9j/4AAQ...",                      │
│   "userPrompt": "Добавь медведя на велосипеде"       │
│ }                                                    │
│                                                      │
│ Content-Type: application/json                       │
│ Max body size: ~2MB (1024px JPEG ≈ 200KB raw,        │
│   base64 ≈ 270KB, с JSON обёрткой ≈ 300KB)           │
└──────────────────┬──────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────┐
│ Backend: AiImageEditService                         │
│                                                      │
│ 1. Проверить JWT → получить userId                   │
│ 2. Проверить баланс монет (≥ 5)                      │
│ 3. Rate limit: max 10 edit запросов в минуту          │
│ 4. Отправить в aggai.ru (multimodal)                 │
│ 5. Получить результат (base64 image)                 │
│ 6. Списать 5 монет с баланса                         │
│ 7. Записать usage (tokens, cost, userId)             │
│ 8. Вернуть base64 результата клиенту                 │
└──────────────────┬──────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────┐
│ Backend → aggai.ru: multimodal request              │
│                                                      │
│ ⚠️ ПРОБЛЕМА: aggai.ru сейчас НЕ поддерживает        │
│ multimodal input (поле input — только строка)        │
│                                                      │
│ НУЖНО: aggai.ru должен поддержать OpenAI-style       │
│ content массив с типами text + image_url:            │
│                                                      │
│ POST /api/functions/apiConversationMessages           │
│ {                                                    │
│   "action": "send",                                  │
│   "conversation_id": "...",                          │
│   "content": [                                       │
│     {                                                │
│       "type": "image_url",                           │
│       "image_url": {                                 │
│         "url": "data:image/jpeg;base64,/9j/4A..."    │
│       }                                              │
│     },                                               │
│     {                                                │
│       "type": "text",                                │
│       "text": "Edit: добавь медведя..."              │
│     }                                                │
│   ]                                                  │
│ }                                                    │
│                                                      │
│ Ответ: images в base64 (как сейчас)                  │
└──────────────────┬──────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────┐
│ Backend → iOS: ответ                                │
│                                                      │
│ {                                                    │
│   "resultImageBase64": "iVBORw0KGgo...",             │
│   "coinsUsed": 5,                                    │
│   "coinsRemaining": 95                               │
│ }                                                    │
│                                                      │
│ Или ошибка:                                          │
│ { "error": "insufficient_coins",                     │
│   "message": "Недостаточно монет" }                  │
└─────────────────────────────────────────────────────┘
```

---

## 4. Задачи для Таймураза (бэкенд)

### 4.1. Запросить/добавить multimodal support в aggai.ru

**Проблема:** `apiConversationMessages` принимает `input` только как строку. Для image editing нужен массив `content` с типами `text` + `image_url` (base64 data URI).

**Текущий формат (работает):**
```json
{
  "action": "send",
  "conversation_id": "...",
  "input": "просто текст"
}
```

**Нужный формат (НЕ работает, возвращает `input.substring is not a function`):**
```json
{
  "action": "send",
  "conversation_id": "...",
  "content": [
    {"type": "image_url", "image_url": {"url": "data:image/jpeg;base64,..."}},
    {"type": "text", "text": "Edit this image: ..."}
  ]
}
```

**Варианты решения:**
1. **Попросить aggai.ru добавить поддержку `content` массива** (OpenAI-compatible формат) — идеальный вариант
2. **Добавить в aggai.ru поле `images`** — массив base64 строк для multimodal input
3. **Обходной путь: бэкенд QRMe сам вызывает OpenRouter/Google AI напрямую** для image edit, минуя aggai.ru

### 4.2. Создать backend endpoint `POST /v3/ai/edit-image`

**Файл:** `backend/backend/src/app-modules/ai-bots/controllers/ai-image-edit.controller.ts` (новый)

```typescript
@Post('v3/ai/edit-image')
@UseGuards(JwtAuthGuard)
async editImage(
  @CurrentUser() user: UserEntity,
  @Body() dto: { imageBase64: string; userPrompt: string }
) {
  // 1. Проверить баланс монет
  // 2. Rate limit (10 req/min per user)
  // 3. Создать conversation в aggai (model: gemini-2.5-flash-image)
  // 4. Отправить multimodal message (image + text)
  // 5. Получить результат
  // 6. Списать монеты
  // 7. Записать usage
  // 8. Вернуть результат
}
```

**Зависимости:**
- `AggaiShimAdapter` — добавить метод `sendMultimodalMessage()` с поддержкой `content` массива
- Монеты — нужна таблица `user_coins` (balance, transactions) или использовать существующую систему

### 4.3. Добавить модель `gemini-2.5-flash-image` в сервис `QRmodels`

Сейчас в `QRmodels` нет image editing модели. Нужно добавить:

| Модель | Сервис | Тип | Цена |
|--------|--------|-----|------|
| `custom/google/gemini-2.5-flash-image` | QRmodels | image-to-image | in=$0.60 out=$5.00 |

Эта модель уже есть в `classic_models`, нужно добавить в `QRmodels` через aggai.ru admin panel.

### 4.4. Система монет (если не реализована)

```sql
CREATE TABLE user_coins (
  user_id UUID PRIMARY KEY REFERENCES users(identifier),
  balance INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE coin_transactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(identifier),
  amount INTEGER NOT NULL,  -- положительный = пополнение, отрицательный = списание
  type VARCHAR(20) NOT NULL,  -- 'topup', 'ai_edit', 'ai_chat', 'ai_image_gen'
  description TEXT,
  metadata JSONB,  -- {model, prompt, cost_usd, tokens}
  created_at TIMESTAMP DEFAULT NOW()
);
```

---

## 5. Модели для image editing в aggai.ru

Проверено 5 апреля 2026. Модели с поддержкой image input + image output:

| Модель ID | Сервис | Input | Output | Цена за запрос |
|-----------|--------|-------|--------|---------------|
| `custom/google/gemini-2.5-flash-image` | classic_models | image+text | image+text | ~$0.04 |
| `custom/google/gemini-3.1-flash-image-preview` | classic_models, QRmodels | image+text | image+text | ~$0.003 |
| `custom/google/gemini-3-pro-image-preview` | classic_models | image+text | image+text | ~$0.024 |
| `custom/openai/gpt-5-image` | classic_models | image+text | image+text | ~$0.20 (дорого) |
| `custom/openai/gpt-5-image-mini` | classic_models | image+text | image+text | ~$0.05 |

**Рекомендация:** `gemini-2.5-flash-image` — лучший баланс цена/качество. Проверен через OpenRouter — реально редактирует (не генерирует заново).

**Важно:** модель `gemini-3.1-flash-image-preview` через aggai.ru НЕ работает для editing — aggai передаёт `input` как текст, и модель генерирует новое изображение вместо редактирования. Для реального editing нужен multimodal input.

---

## 6. Тестовые данные

### Баланс aggai.ru
- $98.62 на 5 апреля 2026
- API key: `uai_in_wlTrYaBWU8ox1d88855bdQpTmdKc8hBY`

### OpenRouter (временный, для MVP)
- API key: `sk-or-v1-668...ae83`
- Проверен: image editing работает с `gemini-2.5-flash-image`
- Cost: ~$0.04 за edit

### Проверенные запросы

**aggai.ru — текст (работает):**
```bash
curl -X POST https://aggai.ru/api/functions/apiConversationMessages \
  -H "Authorization: Bearer $AGGAI_KEY" \
  -H "Content-Type: application/json" \
  -d '{"action":"send","conversation_id":"...","input":"Привет"}'
```

**aggai.ru — multimodal (НЕ работает, нужна доработка):**
```bash
# Возвращает: {"error":{"code":"invalid_request","message":"Missing input"}}
curl -X POST https://aggai.ru/api/functions/apiConversationMessages \
  -H "Authorization: Bearer $AGGAI_KEY" \
  -H "Content-Type: application/json" \
  -d '{"action":"send","conversation_id":"...","content":[{"type":"image_url","image_url":{"url":"data:image/jpeg;base64,..."}},{"type":"text","text":"Edit..."}]}'
```

**OpenRouter — multimodal (работает):**
```bash
curl -X POST https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer $OPENROUTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"google/gemini-2.5-flash-image","messages":[{"role":"user","content":[{"type":"image_url","image_url":{"url":"data:image/jpeg;base64,..."}},{"type":"text","text":"Edit..."}]}]}'
# Ответ: message.images[0].image_url.url = "data:image/png;base64,..."
```

---

## 7. Порядок перехода с OpenRouter на aggai.ru/бэкенд

### Шаг 1: aggai.ru добавляет multimodal support
- Поле `content` как массив `[{type, image_url/text}]` вместо `input` строки
- Или отдельное поле `images` для base64

### Шаг 2: Бэкенд — новый endpoint
- `POST /v3/ai/edit-image` с JWT auth
- Принимает `imageBase64` + `userPrompt`
- Проксирует в aggai.ru с multimodal content
- Проверяет баланс, списывает монеты, пишет usage

### Шаг 3: iOS — переключить на бэкенд
- Заменить `QRmeAIEditService` — вместо прямого вызова OpenRouter вызывать `POST /v3/ai/edit-image` через Moya (endpoint `AIEditImage` уже создан)
- Убрать захардкоженный OpenRouter ключ
- Убрать ATS exception из Info.plist

### Шаг 4: Удалить временные компоненты
- Upload сервер на content-zavod (`/opt/upload-server.py`) — не нужен
- OpenRouter API key — не нужен
- ATS exception для 151.247.197.68 — не нужен

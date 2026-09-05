# RAG Workshop — День 2: Асистент з тренувань (Realistic End-to-End)

RAG-система над **власним корпусом тренувань**: таблиці програми 8.0, лог силових тестів,
програми 3.0 і 4.0 у PDF, протокол антикрихкості.

Ноутбук — форк воркшопу «асистент ріелтора», перенацілений з синтетичних даних агентства
нерухомості на реальні особисті дані. Ланцюг той самий:
**скан `data/` → chunking → embeddings → ChromaDB → retrieval → промпт → відповідь із джерелами**.

**Що розбираємо:**
- **Реальні формати даних**: CSV (експорт з Google Sheets), Markdown, PDF
- **ChromaDB у режимі персистентності** — індекс зберігається між сеансами
- **Метадані для retrieval**: `source` для фільтрації й цитування (+ `kind`, `day`, `week`, `date`, `page`)
- **Стабільна індексація**: `upsert` зі стійкими ID без дублікатів при повторному запуску
- **Маршрутизація контексту**: LLM-роутер `where` обмежує пошук до потрібних файлів
- **Історія діалогу**: чат у межах сесії з контекстом попередніх реплік
- **Промпт-інжиніринг**: цитування джерел, стійкість до галюцинацій, структурована відповідь
- **Evaluation**: золотий набір і метрики якості (coverage джерел, судження LLM)

---

## Корпус (`data/`)

| Файл | Тип | Чанків | Що всередині |
|---|---|---|---|
| `Training Program-8.0-Day 1.csv` | CSV | 10 | Push press, pull-ups, lateral squat, zercher deadlift — 10 тижнів |
| `Training Program-8.0-Day 2.csv` | CSV | 10 | Depth jump, back squat, push-ups, curls, кардіо — 10 тижнів |
| `Training Program-8.0-Day 3.csv` | CSV | 10 | Power clean, close grip bench, sit-ups, chest support row — 10 тижнів |
| `Strength tests Training Program 8.0.csv` | CSV | 8 | Лог силових тестів: вправа, результат, повтори, дата, власна вага |
| `Training Program 3.0.pdf` | PDF | 3 | Склад програми 3.0 по днях + позначення періодів |
| `Training Program 4.0.pdf` | PDF | 4 | Склад програми 4.0 по днях + позначення |
| `Antifragility protocol.md` | MD | 2 | Протокол профілактики травм (спина, коліна, плечі) |

**Разом: 47 чанків.**

### Чому CSV тут — окремий випадок

Таблиці програми 8.0 — це експорт із Google Sheets, а не нормалізована таблиця:

```
Day 1,,,,,,,,,,,
,,,,,,,,,,,
Week,Push press,,Pull-ups,,Lateral squat,,Zercher deadlift,,Cardio,,Notes
1,1,30kg x 6,1,10kg x 3,1,10,1,30kg x 10,,,
,2,35kg x 6,2,10kg x 3,2,10,2,30kg x 10,,,
10 Nov,3,40kg x 6,3,10kg x 3,3,10,3,35kg x 10,,,
```

- назва вправи в заголовку займає **дві колонки**: номер підходу + результат;
- **тиждень** стоїть у колонці `Week` лише в першому рядку блоку, **дата** — у третьому;
- окремий рядок (`,2,35kg x 6,...`) **не має ні тижня, ні дати, ні назв вправ**.

Правило воркшопу «один рядок = один документ» дало б тут сміття. Замість нього:

1. **Карта заголовків** — непорожня клітинка в колонці `i` «володіє» колонками
   `i..(наступна непорожня − 1)`. Одне це правило покриває обидва макети: пари
   «номер підходу + результат» (Day 1) і «значення + порожньо» (Day 2).
2. **Тижневі блоки** — новий блок починається, коли в колонці `Week` ціле число;
   непорожнє нечислове значення там же (`10 Nov`) — це дата блоку.
3. **Один чанк = один тиждень**:

```
Training Program 8.0 — Day 1 — Week 3 (24 Nov)
Push press: 1) 40kg x 6; 2) 40kg x 6; 3) 45kg x 6; 4) 45kg x 6
Pull-ups: 1) 16kg x 3; 2) 16kg x 3; 3) 16kg x 3
Lateral squat: 1) 8kg x 10; 2) 12kg x 10; 3) 12kg x 10
Zercher deadlift: 1) 40kg x 10; 2) 45kg x 10; 3) 45kg x 10
Cardio: —
Notes: —
```

Лог силових тестів має іншу форму — там працює звичне «один рядок = один документ»
(після пропуску преамбули з інструкцією).

---

## Необхідні credentials

**OpenAI API key** (`OPENAI_API_KEY`)
- Отримати тут: [https://platform.openai.com/](https://platform.openai.com/)
- Локально: `.env` у корені репозиторію (шаблон — `.env.example`)
- На Colab: додайте через Secrets (🔑 icon)
- Потрібен активний billing на акаунті

Telegram-бот у цьому ноутбуці **відсутній** — секцію прибрано разом із її токеном.

---

## Відкрити у Google Colab

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/ingvar-goryainov/CrashCourse_Building_RAG_system/blob/main/Day_2/rag_workshop_02_training_assistant.ipynb)

Пряме посилання:
https://colab.research.google.com/github/ingvar-goryainov/CrashCourse_Building_RAG_system/blob/main/Day_2/rag_workshop_02_training_assistant.ipynb

---

## Вміст (День 2)

- `rag_workshop_02_training_assistant.ipynb` — **основний ноутбук**
- `data/` — корпус документів для індексації (CSV, Markdown, PDF)
- `chroma_data/training_assistant/` — вектор-індекс (генерується, у git не потрапляє)
- `AdvancedRAG_pipeline.png` — загальна схема пайплайну
- `requirements.txt` — залежності воркшопу
- `README.md` — цей файл
- `rag_workshop_02_realtor_assistant.ipynb` — оригінал воркшопу, **більше не запускається** (див. нижче)

---

## Швидкий старт (Colab)

1. **Відкрийте** notebook у Colab через бейдж вище.
2. **Збережіть копію**: `File → Save a copy in Drive`.
3. **Додайте secret** (ліва панель → 🔑 key icon): `OPENAI_API_KEY`.
4. **Запустіть комірки** послідовно (Shift+Enter):
   - **Setup** — встановлення бібліотек і клонування репозиторію в `/content`
   - **Dry run** — парсинг `data/` **без** звернень до OpenAI
   - **Індексація** — embeddings і `upsert` у Chroma
   - **Retrieval** / **Generation** — приклади пошуку й відповідей із джерелами
   - **Router**, **Історія**, **Evaluation** — додаткові кейси

---

## Локальний запуск

```bash
# у корені репозиторію
python -m venv .venv
source .venv/bin/activate          # На Windows: .venv\Scripts\activate

pip install openai chromadb python-dotenv pandas pymupdf
# (Day_2/requirements.txt теж підійде — це надмножина, там ще python-telegram-bot,
#  який цьому ноутбуку не потрібен)

cp .env.example .env               # і вписати свій OPENAI_API_KEY

jupyter lab Day_2/rag_workshop_02_training_assistant.ipynb
```

### Перевірка без витрат на API

Комірка **«Dry run»** проганяє всі обробники файлів і друкує, що саме потрапить в індекс —
кількість чанків за файлами і приклад тижневого блоку — **не звертаючись до OpenAI**.
Зручно побачити помилку розбору до того, як витрачати токени.

Перевірено на цьому корпусі без API-ключа:

```
47 чанків       (30 тижневих блоків + 8 записів + 4 PDF + 2 MD)
повторний прогін індексації: 47 → 47      ← стабільні id не дублюють записи
n_results       k=3/6/10 дотримується
where           source / $and(kind, week) / $in — повертають очікувані рядки
```

### Повна перебудова індексу

У комірці створення колекції: `FORCE_REBUILD = True` — видаляє колекцію
`training_corpus_v1` і індексує `data/` з нуля. Потрібно після зміни логіки чанкінгу
або складу метаданих.

---

## Як закриті вимоги завдання

| # | Вимога | Де в ноутбуці |
|---|---|---|
| 1 | Обхід теки → chunking → embeddings → ChromaDB (`upsert`, стабільні id) | `iter_corpus_files`, `chunker`, `embed_many`, `upsert_batches`; id виду `…::w3` / `…::r4` / `…::p1::c0` |
| 2 | Метадані з `source` для цитування і `where` | `_str_meta({"source": …, "kind": …, "day": …, "week": …})` |
| 3 | `collection.query` з `query_embeddings` і параметром `k` | `retrieve(query, k=6, where=None)` |
| 4 | Промпт «лише з контексту» + блок «Джерела:» у форматі `[source=…]` | `PROMPT_TEMPLATE`, `build_context`, `rag_answer` |
| 5 | Відтворювана логіка | цей `.ipynb`; `FORCE_REBUILD` для чистої перебудови |

**Ідемпотентність:** id детерміновані (шлях файлу + позиція чанка), тож повторний прогін
індексації робить `upsert` поверх тих самих записів — `collection.count()` не росте.

---

## Troubleshooting

- **`OPENAI_API_KEY is missing` / `OPENAI_API_KEY: ✗ ВІДСУТНІЙ`**
  - **Локально**: `cp .env.example .env` і вписати ключ; `.env` має лежати в **корені репозиторію**
  - **На Colab**: додайте до Secrets (🔑 icon, ліва панель)
  - Без ключа працює лише комірка **Dry run**

- **`ModuleNotFoundError: No module named 'chromadb'`**
  - На Colab: запустіть комірку **Setup**
  - Локально: перевірте, що активовано саме `.venv` (`which python`)

- **`import chromadb` падає з `No module named 'tokenizers'` або `opentelemetry.sdk`**
  - Типово для «великого» conda-оточення, де вже стоять `crewai` / `embedchain` /
    `transformers` із конфліктними пінами: chromadb 0.5.x будує default embedding function
    ще на етапі імпорту й тягне `tokenizers`.
  - Лікується не доустановкою пакетів у те саме оточення, а **чистим venv** (див. «Локальний запуск»).

- **Відповідь без блоку «Джерела:» або з вигаданим шляхом**
  - Перевірте, що `build_context` додає заголовок `[source=…]` перед кожним фрагментом
  - Роутер фільтрує вибір моделі по `ALLOWED_SOURCES`, тому шлях поза індексом не пройде

- **`collection.count()` росте з кожним прогоном**
  - Значить, id перестали бути стабільними — перевірте `_safe_id_base` і суфікси
    (`::w{week}`, `::r{row}`, `::p{page}::c{chunk}`)

---

## Ноутбук ріелтора

`rag_workshop_02_realtor_assistant.ipynb` лишено в репозиторії як оригінал воркшопу, але він
**більше не запускається**: його корпус (`listings.csv`, `clients.csv`, `contracts/`,
`taxes_2026.md` тощо) і `use_case_realtor_assistant.md` прибрано з `data/` разом із переходом
на тренувальний корпус.

Відновити ті файли можна з коміту `90cb144`:

```bash
git checkout 90cb144 -- Day_2/data Day_2/use_case_realtor_assistant.md
```

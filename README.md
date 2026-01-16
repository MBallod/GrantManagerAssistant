# Grant RAG MVP

RAG-система поддержки экспертизы благотворительных заявок.  
Проект предназначен для анализа текстовых заявок объёмом **4–7 страниц**, формирования **экспертных выводов и рекомендаций по доработке**, а также оценки **доверия к результату ИИ** с помощью обучаемого калибратора.

---

## Цели проекта
- Сократить среднее время экспертизы заявки **на ≥30%**
- Повысить стандартизацию экспертной оценки
- Обеспечить объяснимость результатов (цитирование источников ≥99%)
- Поддержать human-in-the-loop, не автоматизируя финальное решение

---

## Ключевые возможности
- RAG-анализ текстовых заявок (PDF/DOCX)
- Генерация:
  - оценки по критериям программы
  - рекомендаций по доработке заявки
- Обязательное цитирование источников (chunk_id)
- Режим *insufficient evidence* при нехватке данных
- Калибратор доверия (`p_ok`, `risk_score`)
- Полная воспроизводимость inference-запусков

---

## Архитектура (кратко)
- **FastAPI** — inference API и orchestration
- **SentenceTransformers** — эмбеддинги
- **Qdrant** — vector search (ANN)
- **LLM** — GPT-4 / GigaChat / LLaMA / Qwen (inference-only)
- **PostgreSQL** — Core DWH (сущности, версии, audit)
- **MinIO / S3** — Data Lake (файлы и артефакты)
- **Калибратор доверия** — обучаемая ML-модель поверх RAG

---

## Структура проекта
```
grant-rag-mvp/
├── README.md
│   └── Описание проекта, запуск, архитектура
│
├── .env.example
│   └── Переменные окружения
│
├── docker-compose.yml
│   └── API + Postgres + Qdrant + MinIO
│
├── pyproject.toml
│   └── Зависимости и инструменты
│
├── docs/
│   ├── PRD.md
│   │   └── Краткий PRD и продуктовые метрики
│   ├── DATA Processing.md
│   │   └── Структура хранения и обработки данных
│   └── ML Manifest.md
│       └── Описание ML части
│
├── src/
│   ├── main.py
│   │   └── FastAPI entrypoint и роуты
│   ├── settings.py
│   │   └── Конфигурация из env
│   ├── schemas.py
│   │   └── Pydantic DTO и схемы
│   │
│   ├── rag/
│   │   ├── pipeline.py
│   │   │   └── Orchestrator: embed → retrieve → prompt → LLM → validate → persist
│   │   ├── chunking.py
│   │   │   └── Chunking (size / overlap / sections)
│   │   ├── retrieval.py
│   │   │   └── Qdrant search (top-k, filters, dedup)
│   │   ├── prompts.py
│   │   │   └── Шаблоны промптов + формат JSON
│   │   └── validation.py
│   │       └── Citation validity + coverage + schema checks
│   │
│   ├── adapters/
│   │   ├── llm.py
│   │   │   └── Клиент LLM (retries / timeouts)
│   │   ├── embedder.py
│   │   │   └── SentenceTransformers embeddings (CPU / GPU)
│   │   ├── vector_db.py
│   │   │   └── Qdrant client (search / upsert)
│   │   ├── dwh.py
│   │   │   └── Postgres: applications / inference_runs / versions
│   │   └── lake.py
│   │       └── S3 / MinIO: raw / parsed / chunks / runs / models
│   │
│   └── calibrator/
│       ├── features.py
│       │   └── Фичи качества (retrieval / generation / input)
│       ├── predict.py
│       │   └── Risk score inference
│       └── train.py
│           └── Offline-обучение калибратора
│
├── scripts/
│   ├── ingest_to_lake.py
│   │   └── Загрузка PDF / DOCX в raw + manifest
│   ├── parse_and_chunk.py
│   │   └── raw → parsed + chunks
│   ├── upsert_index.py
│   │   └── chunks → Qdrant + index_version
│   ├── eval_retrieval.py
│   │   └── Precision@k, nDCG@k
│   ├── bootstrap.sql
│   │   └── Минимальные таблицы Postgres
│   └── bootstrap_env.sh
│       └── Buckets + SQL + smoke checks
│
└── tests/
    ├── test_validation.py
    │   └── validity / coverage / schema
    ├── test_chunking.py
    │   └── стабильность chunking
    └── test_features.py
        └── фичи калибратора
```
---

## Поток данных (inference)
1. Заявка (ID или текст) поступает в `/analyze`
2. Генерация эмбеддингов запроса
3. Retrieval top-k чанков из Qdrant
4. Формирование контекста и prompt
5. Генерация ответа LLM (JSON)
6. Валидация структуры и цитирования
7. Расчёт `risk_score` калибратором
8. Сохранение метаданных в DWH и артефактов в Data Lake

---

## Калибратор доверия
Обучаемая ML-модель, предсказывающая вероятность того,  
что эксперт примет результат ИИ без существенных правок.

**Используемые признаки:**
- median similarity score
- coverage
- citation validity
- длина и структура ответа
- latency и параметры retrieval

**Выход:**
- `p_ok` — вероятность принятия
- `risk_score = 1 - p_ok`

---

## Метрики качества

### Retrieval (offline)
- Precision@k ≥ 0.6
- nDCG@k ≥ 0.7

### RAG (online)
- Coverage ≥ 80%
- Citation validity ≥ 99%
- Insufficient evidence rate: 10–25%

### Business
- Time-to-review ↓ ≥ 30%
- Override rate ≤ 40%

---

## Data Storage
- **Data Lake (S3/MinIO)**  
  RAW файлы, parsed текст, чанки, RAG-трейсы, ML-артефакты
- **Core DWH (PostgreSQL)**  
  Заявки, эксперты, критерии, решения, версии, inference_runs
- **Vector DB (Qdrant)**  
  Эмбеддинги и ANN-индексы

---

## Мониторинг и качество данных
- **Great Expectations** — контроль качества parsed/chunks/staging
- **OpenTelemetry** — трассировка inference
- **Uptrace / Grafana** — latency, error rate, coverage

---

## Запуск (MVP)

```bash
cp .env.example .env
docker compose up -d
```
## API будет доступен по адресу:
```bash
http://localhost:8000/docs
```

## Ограничения MVP

- LLM и embedding-модели не обучаются
- Нет автоматического принятия решений
- Batch-пайплайны запускаются скриптами

## Назначение проекта

- Проект ориентирован на MVP и исследовательскую работу (ВКР) с акцентом на архитектуру, воспроизводимость и объяснимость ML-системы.
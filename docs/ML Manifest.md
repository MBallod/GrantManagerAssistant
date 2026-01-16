# ML Manifest — Grant RAG System  
*(ML System Design / PRD-style)*

## 1. Scope

ML-часть системы поддерживает экспертизу заявок объёмом **4–7 страниц** через связку **RAG + LLM**  
и оценивает надёжность результата с помощью **калибратора доверия**.

LLM и embedding-модели **не обучаются** в проекте;  
единственный обучаемый компонент — **risk / trust calibrator**.

---

## 2. Models Used

### 2.1 Embedding Model (Retrieval)

- **Модель:** SentenceTransformers bi-encoder  
  (например, семейство `all-MiniLM` или аналог)
- **Режим:** inference-only
- **Назначение:** преобразование запросов и чанков в векторы для семантического поиска
- **Выход:** embedding-вектор + `chunk_id`

---

### 2.2 LLM (Generation)

- **Модели:**
  - GPT-4 / GPT-4o (API)
  - GigaChat (закрытый контур, русский язык)
  - LLaMA-3 / Qwen (локально или собственный API)
- **Назначение:** генерация структурированного **JSON-ответа**, включающего:
  - оценку по критериям,
  - рекомендации по доработке заявки,
  - ссылки на источники
- **Обязательное требование:** все утверждения содержат ссылки на `chunk_id`

---

### 2.3 Risk Calibrator (Confidence Scoring)

> **Единственный обучаемый ML-компонент системы**

- **Модель:**
  - baseline: Logistic Regression
  - production: Gradient Boosting (LightGBM / XGBoost / CatBoost)
  - + калибровка вероятностей
- **Назначение:** предсказать  
  `p_ok` — вероятность того, что эксперт примет результат без существенных правок
- **Выход:**  
  `risk_score = 1 - p_ok`
- **Использование:**
  - флаг *low confidence*,
  - включение режимов деградации,
  - приоритизация ручной проверки

---

## 3. Algorithms

### 3.1 Retrieval

- ANN-индекс: **HNSW** (Qdrant)
- `top-k`: **5–15**
- опционально: **re-ranking cross-encoder** (желательно GPU)

---

### 3.2 Generation

- versioned prompt templates
- constrained output:
  - JSON
  - schema validation
- retries при:
  - невалидном JSON,
  - слабых или отсутствующих цитированиях

---

### 3.3 Calibration

- табличная классификация / регрессия вероятности
- `class_weight` / threshold tuning (при дисбалансе)
- calibration:
  - Platt scaling
  - Isotonic regression

---

## 4. Data Flow

### 4.1 Offline Data (DWH + Lake)

- **Data Lake:**
  - raw PDF / DOCX
  - parsed text
  - chunks
  - `rag_runs` traces
- **Core DWH (Postgres):**
  - `applications`
  - `criteria`
  - `expert_reviews`
  - `inference_runs`
  - `versions`

---

### 4.2 Online Inference Flow

1. вход: `application_id` или текст заявки → FastAPI  
2. `embed(query)` → Vector DB search (`top-k`) → контекст  
3. LLM generate (JSON) → validation (citations / coverage / schema)  
4. feature builder → calibrator → `p_ok`  
5. persist:
   - DWH (`inference_run`)
   - Lake (`rag_run` artifacts)

---



## 5. Training Stage

Обучается **только калибратор доверия** на данных: features(inference) + labels(expert override)
- Split: 70/30 или k-fold
- Периодичность:
  - раз в спринт / месяц
  - при накоплении ≥ **200** новых размеченных inference-run’ов
- Артефакты модели:
  - Data Lake: `models/calibrator/vX`
  - версия фиксируется в DWH

---

## 6. Metrics

### 6.1 Retrieval Quality (offline)

- Precision@k ≥ **0.6**
- nDCG@k ≥ **0.7**
- (опционально) Recall@k ≥ **0.8**

---

### 6.2 RAG Quality

- Coverage ≥ **80%**
- Citation validity ≥ **99%**
- Insufficient evidence rate: **10–25%**
- Hallucination rate (на экспертной выборке) ≤ **5%**

---

### 6.3 Calibrator Quality (offline)

- ROC-AUC ≥ **0.7**
- Brier score ↓
- ECE ↓
- Lift:
  - top-30% по `p_ok`
  - override rate ниже среднего ≥ **10 п.п.**

---

### 6.4 Ops / Serving

- p95 latency ≤ **30s**
- error rate < **1%**
- availability ≥ **99%** (MVP)

---

## 7. Quality Monitoring

### 7.1 Online (без разметки)

- median similarity score  
  *(алерт при падении > 15% WoW)*
- coverage
- citation validity
- insufficient evidence rate
- latency p50 / p95
- error rate
- распределение `p_ok`:
  - PSI < 0.2 — норма
  - PSI ≥ 0.2 — дрейф

---

### 7.2 Offline (с разметкой)

- регулярный пересчёт Precision@k, nDCG@k
- выборочная проверка:
  - галлюцинаций
  - корректности рекомендаций  
  *(n = 50–100 кейсов)*

---

### 7.3 Response Actions

- citation validity < 99% → блок / режим деградации
- insufficient evidence > 40% → сигнал “новый домен / сломан retrieval”
- p95 latency > 30s → алерт + профилирование pipeline

---

## 8. Serving

### 8.1 Inference Service (FastAPI)

- Endpoint: `/analyze`
- Внутри:
  - orchestration pipeline
  - validation
  - calibrator inference
- Параметры:
  - `prompt_version`
  - `retrieval_config_id`
  - `k`
  - `llm_model`

---

### 8.2 Embeddings

- **Вариант A (MVP):** embeddings внутри сервиса (CPU/GPU)
- **Вариант B (нагрузка):** отдельный `embed-service`

---

### 8.3 Calibrator

- Загружается при старте сервиса:
  - по `calibrator_version`
  - или `latest_production`
- Inference на CPU за миллисекунды  
  *(не влияет существенно на p95)*

---

### 8.4 Vector DB

- Qdrant как отдельный сервис
- Обновление индекса:
  - batch scripts
  - ETL jobs

---

## 9. Versioning & Reproducibility

Каждый `inference_run` фиксирует:

- `prompt_version`
- `embedding_version`
- `index_version`
- `retrieval_config_id`
- `llm_model_version`
- `calibrator_version`

Артефакты запуска сохраняются в Data Lake:
```
rag_runs/run_id=...
├── prompt_snapshot
├── retrieved_chunks.json
├── raw_output.json
└── validated_output.json
```

## 10. Minimal Tooling Stack

- Python + FastAPI + Pydantic
- SentenceTransformers + PyTorch
- Qdrant (vector store)
- PostgreSQL (Core DWH)
- MinIO / S3 (Data Lake)
- OpenTelemetry + Uptrace + Grafana
- MLflow (опционально, для калибратора)


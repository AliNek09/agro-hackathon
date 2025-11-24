# VendorSync - AI-сверка платёжных документов для B2B

## 🎯 Executive Summary

**Проблема**: Малый бизнес в Узбекистане тратит 15-20 часов/месяц на ручную сверку счетов, актов и платёжных поручений с контрагентами. Ошибка в одной цифре = задержка платежа на недели.

**Решение**: Telegram-бот с AI, который автоматически распознаёт документы, находит расхождения и генерирует акты сверки.

**Результат**:
- 90% экономии времени на сверку
- 0 потерянных документов
- Автоматические напоминания о неоплаченных счетах

---

## 🔍 Problem Deep Dive

### Текущий процесс (manual reconciliation hell)

**Типичный сценарий**:
1. Компания А отправила счёт компании Б на 5 млн сум
2. Компания Б оплатила, но в платёжке написала 4.8 млн (ошибка в реквизитах)
3. Компания А видит платёж, но не может сопоставить со счётом
4. В конце месяца: бухгалтер А и бухгалтер Б тратят 3 часа на Zoom/телефон для сверки
5. Находят ошибку → Б делает доплату → ещё неделя
6. **Итого**: 1 месяц цикл вместо недели

### Pain Points:
1. **Ручной труд**: Excel с сотнями строк, VLOOKUP не работает из-за разных форматов
2. **Человеческие ошибки**: Опечатки в номерах счетов, датах, суммах
3. **Разнообразие форматов**: PDF, Word, сканы, фото, разные шаблоны
4. **Коммуникационный overhead**: Звонки, переписки, потерянные документы в email
5. **Юридические риски**: Без акта сверки сложно доказать долг в суде

### Market Size:
- **250K+ SMB** в Узбекистане
- **5-10 контрагентов** у среднего бизнеса
- **20 часов/месяц** × $5/час = $100/месяц потерь
- **TAM**: 250K × $100 = $25M/месяц потерянного времени

---

## 💡 Solution Architecture

### Core Workflow

```
┌─────────────────────────────────────────────────────┐
│  1. Document Upload (Telegram Bot)                 │
│  - Фото/PDF счёта, акта, платёжки                  │
│  - Автоматическая классификация типа документа     │
└─────────────────┬───────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────┐
│  2. OCR + Entity Extraction (AI)                   │
│  - PaddleOCR для текста                            │
│  - GPT-4o-mini для структурирования:               │
│    • Номер документа                               │
│    • Дата                                          │
│    • Контрагент (ИНН, название)                    │
│    • Сумма                                         │
│    • Назначение платежа                            │
└─────────────────┬───────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────┐
│  3. Smart Matching (Fuzzy Logic)                   │
│  - Связывание: счёт ↔ акт ↔ платёжка              │
│  - Fuzzy matching для опечаток:                    │
│    • "ООО Компания" vs "Компания ООО"             │
│    • "5000000" vs "5 000 000.00"                   │
│    • "01.11.2025" vs "01/11/25"                    │
└─────────────────┬───────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────┐
│  4. Discrepancy Detection                          │
│  - Найти несоответствия:                           │
│    ✗ Счёт на 5M, оплата 4.8M → разница 200K       │
│    ✗ Акт №123 нет соответствующей платёжки        │
│    ✓ Счёт №456 полностью оплачен                  │
└─────────────────┬───────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────┐
│  5. Reconciliation Report                          │
│  - Генерация акта сверки (Excel/PDF)              │
│  - Отправка контрагенту через Telegram            │
│  - Автоподпись через ЭЦП (опционально)            │
└─────────────────────────────────────────────────────┘
```

---

## 🛠️ Technical Implementation

### Tech Stack

**Backend**:
- **FastAPI** (Python 3.11+) - REST API
- **PostgreSQL** - хранение документов и сущностей
- **Neo4j** (опционально) - graph-based связи между документами
- **Redis** - кеш для OCR результатов

**AI/ML**:
- **PaddleOCR** - распознавание текста (Cyrillic support out of box)
- **OpenAI GPT-4o-mini** - извлечение структурированных данных
- **FuzzyWuzzy / RapidFuzz** - fuzzy string matching
- **spaCy** - NER для узбекских/русских организаций

**Frontend**:
- **Telegram Bot API** - основной интерфейс
- **React Dashboard** - веб-панель для бухгалтеров

**Integration**:
- **Bank API** (если доступно) - автоматическая загрузка платёжек из выписки
- **1C API** (будущее) - синхронизация с бухгалтерским ПО

---

### System Architecture

```
┌──────────────┐
│ Telegram Bot │ ← Пользователь загружает документы
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  FastAPI     │
│  /upload     │
│  /match      │
│  /report     │
└──────┬───────┘
       │
   ┌───┴────┐
   ▼        ▼
┌───────┐ ┌──────────┐
│ Redis │ │PostgreSQL│
│(cache)│ │(entities)│
└───────┘ └────┬─────┘
               │
               ▼
        ┌──────────────┐
        │ ML Pipeline  │
        │ - OCR        │
        │ - Extraction │
        │ - Matching   │
        └──────────────┘
```

---

### Core Algorithms

#### 1. OCR + Entity Extraction

```python
import paddleocr
from openai import OpenAI

# Step 1: OCR
ocr = paddleocr.PaddleOCR(lang='cyrillic')
result = ocr.ocr(image_path)
full_text = "\n".join([line[1][0] for line in result[0]])

# Step 2: Entity extraction
client = OpenAI()
prompt = f"""
Extract structured data from this Uzbekistan business document.

Document text:
{full_text}

Return JSON:
{{
  "doc_type": "invoice|act|payment",
  "doc_number": "...",
  "date": "YYYY-MM-DD",
  "counterparty": {{
    "name": "...",
    "inn": "..."
  }},
  "amount": float,
  "currency": "UZS",
  "description": "..."
}}
"""

response = client.chat.completions.create(
    model="gpt-4o-mini",
    response_format={"type": "json_object"},
    messages=[{"role": "user", "content": prompt}]
)

structured_data = json.loads(response.choices[0].message.content)
```

#### 2. Fuzzy Matching

```python
from rapidfuzz import fuzz
import pandas as pd

def match_invoice_to_payment(invoices_df, payments_df, threshold=80):
    matches = []
    
    for _, invoice in invoices_df.iterrows():
        for _, payment in payments_df.iterrows():
            # Multi-factor matching
            
            # 1. Amount match (exact or close)
            amount_match = abs(invoice['amount'] - payment['amount']) < 0.01 * invoice['amount']
            
            # 2. Counterparty name fuzzy match
            name_score = fuzz.token_set_ratio(
                invoice['counterparty_name'],
                payment['counterparty_name']
            )
            
            # 3. Date proximity (within 30 days)
            date_diff = abs((invoice['date'] - payment['date']).days)
            date_match = date_diff <= 30
            
            # 4. Description similarity
            desc_score = fuzz.partial_ratio(
                invoice['description'],
                payment['description']
            )
            
            # Combined scoring
            if amount_match and name_score > threshold and date_match:
                confidence = (name_score + desc_score) / 2
                matches.append({
                    'invoice_id': invoice['id'],
                    'payment_id': payment['id'],
                    'confidence': confidence,
                    'amount_diff': invoice['amount'] - payment['amount']
                })
    
    return pd.DataFrame(matches).sort_values('confidence', ascending=False)
```

#### 3. Discrepancy Detection

```python
def find_discrepancies(company_id):
    # Get all documents for company
    invoices = get_invoices(company_id)
    acts = get_acts(company_id)
    payments = get_payments(company_id)
    
    # Match invoices to payments
    matches = match_invoice_to_payment(invoices, payments)
    
    discrepancies = []
    
    # Type 1: Unpaid invoices
    paid_invoice_ids = matches['invoice_id'].unique()
    unpaid = invoices[~invoices['id'].isin(paid_invoice_ids)]
    for _, inv in unpaid.iterrows():
        days_overdue = (datetime.now() - inv['date']).days
        discrepancies.append({
            'type': 'unpaid_invoice',
            'severity': 'high' if days_overdue > 30 else 'medium',
            'invoice_num': inv['number'],
            'amount': inv['amount'],
            'days_overdue': days_overdue
        })
    
    # Type 2: Partial payments
    for _, match in matches.iterrows():
        if match['amount_diff'] > 100:  # More than 100 UZS difference
            discrepancies.append({
                'type': 'partial_payment',
                'severity': 'medium',
                'invoice_num': match['invoice_number'],
                'expected': match['invoice_amount'],
                'received': match['payment_amount'],
                'difference': match['amount_diff']
            })
    
    # Type 3: Payment without invoice (overpayment or wrong recipient)
    matched_payment_ids = matches['payment_id'].unique()
    orphan_payments = payments[~payments['id'].isin(matched_payment_ids)]
    for _, pmt in orphan_payments.iterrows():
        discrepancies.append({
            'type': 'unmatched_payment',
            'severity': 'high',
            'payment_num': pmt['number'],
            'amount': pmt['amount'],
            'date': pmt['date']
        })
    
    return discrepancies
```

---

## 🚀 MVP Roadmap (2 недели)

### Week 1: Core OCR & Extraction

**Day 1-2**: Infrastructure Setup
- FastAPI project structure
- PostgreSQL schema (tables: companies, documents, entities, matches)
- Docker Compose (app, db, redis)
- Telegram Bot registration

**Day 3-4**: OCR Pipeline
- PaddleOCR integration
- Document classification (invoice vs act vs payment)
- Image preprocessing (rotation, denoise)
- Test on 10-20 sample documents

**Day 5-7**: Entity Extraction
- GPT-4o-mini prompt engineering
- Parse JSON responses
- Validation rules (INN format, amount > 0, valid dates)
- Save entities to DB

### Week 2: Matching & Reporting

**Day 8-9**: Matching Algorithm
- Implement fuzzy matching logic
- Test on synthetic dataset (create 50 invoice-payment pairs with intentional mismatches)
- Tune threshold parameters

**Day 10-11**: Telegram Bot UX
- `/start` - registration, company profile
- `/upload` - send document photo → OCR → show extracted data → confirm/edit
- `/counterparties` - list all counterparties
- `/check <counterparty>` - run reconciliation with specific counterparty
- `/report` - generate full reconciliation report

**Day 12-13**: Report Generation
- Excel export with reconciliation table
- PDF act of reconciliation (using template)
- Send via Telegram
- Email notification (optional)

**Day 14**: Testing & Demo
- End-to-end test: upload 20 docs → match → report
- Demo script
- Video recording
- Deployment

---

## 📊 Success Metrics

### User Metrics:
- **Time saved**: Measure 20 hours → 2 hours (90% reduction)
- **Accuracy**: 95%+ correct matching (manual verification needed for edge cases)
- **User satisfaction**: NPS score

### Technical Metrics:
- **OCR accuracy**: 98%+ character recognition rate (Cyrillic)
- **Extraction accuracy**: 90%+ correct entity extraction
- **Matching precision**: 85%+ (fewer false positives)
- **Matching recall**: 95%+ (don't miss real matches)

### Business Metrics:
- **Adoption**: 100 companies using MVP within 3 months
- **Retention**: 70%+ monthly retention
- **Virality**: Counterparties invite each other (network effect)

---

## 💰 Business Model

### Pricing Tiers:

**Free Tier** (MVP):
- 10 documents/month
- 1 counterparty
- Basic matching
- Email support

**Starter** ($20/month):
- 100 documents/month
- 5 counterparties
- Advanced matching with fuzzy logic
- Email + chat support

**Business** ($50/month):
- Unlimited documents
- Unlimited counterparties
- Bank API integration (auto-import payments)
- Priority support
- Custom reports

**Enterprise** ($200+/month):
- Everything in Business
- 1C integration
- Dedicated account manager
- SLA 99.9%
- Custom workflows

### Revenue Projections:
- **Year 1**: 500 paying customers × $30 avg = $15K MRR = $180K ARR
- **Year 2**: 2,000 customers × $35 avg = $70K MRR = $840K ARR
- **Year 3**: 5,000 customers × $40 avg = $200K MRR = $2.4M ARR

---

## 🚧 Risks & Mitigation

### Technical Risks:

**Risk**: OCR quality low for poor-quality scans  
**Mitigation**: Implement image preprocessing (auto-rotation, contrast enhancement); guide users to take good photos

**Risk**: GPT-4o-mini hallucinations (wrong numbers)  
**Mitigation**: Validate extracted data with heuristics; show confidence score; allow manual correction

**Risk**: Fuzzy matching too many false positives  
**Mitigation**: Conservative thresholds; always show confidence score; human-in-the-loop for low confidence

### Business Risks:

**Risk**: Network effect slow to start (both sides need to use)  
**Mitigation**: Single-sided utility: даже если контрагент не использует, компания видит свои неоплаченные счета

**Risk**: Integration with 1C/bank systems complex  
**Mitigation**: MVP works standalone; integrations as Phase 2

**Risk**: Larger player (Karmon AI, Azma) adds this feature  
**Mitigation**: Move fast, build network effect, become de-facto standard

---

## 🎯 Why This Will Win

1. **Unique positioning**: No direct competitor in Uzbekistan for B2B reconciliation
2. **Clear ROI**: 20 hours → 2 hours = $100/month saved vs $20-50 subscription
3. **Network effects**: As more counterparties join, value increases exponentially
4. **Integration opportunity**: Agrobank can offer this to SMB clients → stickiness
5. **Scalable tech**: Once OCR + matching works, can handle unlimited volume

---

## 📚 Sample Data Structures

### Database Schema
```sql
CREATE TABLE companies (
    id UUID PRIMARY KEY,
    name VARCHAR(255),
    inn VARCHAR(9),
    created_at TIMESTAMP
);

CREATE TABLE documents (
    id UUID PRIMARY KEY,
    company_id UUID REFERENCES companies(id),
    type VARCHAR(20), -- 'invoice', 'act', 'payment'
    file_url VARCHAR(500),
    uploaded_at TIMESTAMP,
    ocr_text TEXT,
    status VARCHAR(20) -- 'pending', 'processed', 'matched'
);

CREATE TABLE entities (
    id UUID PRIMARY KEY,
    document_id UUID REFERENCES documents(id),
    doc_number VARCHAR(100),
    doc_date DATE,
    counterparty_name VARCHAR(255),
    counterparty_inn VARCHAR(9),
    amount NUMERIC(15, 2),
    currency VARCHAR(3),
    description TEXT,
    extracted_at TIMESTAMP
);

CREATE TABLE matches (
    id UUID PRIMARY KEY,
    invoice_entity_id UUID REFERENCES entities(id),
    payment_entity_id UUID REFERENCES entities(id),
    confidence_score FLOAT,
    amount_difference NUMERIC(15, 2),
    matched_at TIMESTAMP,
    verified_by_user BOOLEAN DEFAULT FALSE
);
```

---

**Let's automate B2B reconciliation! 🚀**

# RC Desk — Adaptive Reading Comprehension Engine

[![Java 17](https://img.shields.io/badge/Java-17-orange.svg)](https://www.oracle.com/java/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.x-brightgreen.svg)](https://spring.io/projects/spring-boot)
[![MySQL](https://img.shields.io/badge/MySQL-8.0-blue.svg)](https://www.mysql.com/)

**RC Desk** is a production-grade Spring Boot web application designed for competitive exam aspirants (CAT, GMAT, SBI PO, IBPS PO, UPSC) to convert raw articles, URLs, or newspaper clippings into exam-style Reading Comprehension (RC) practice sets with instant evaluation and adaptive difficulty tracking.

<img width="655" height="404" alt="image" src="https://github.com/user-attachments/assets/55b294ea-dc66-4323-a9fa-2512e8585a36" />


---

## Key Engineering Highlights

1. **Resilient SPA & URL Scraping Engine (`Jsoup`):**
   * Bypasses standard web scraping limitations by extracting content through semantic selectors and density scoring.
   * Features specialized handling for Next.js hydration states (`__NEXT_DATA__`) to extract articles from complex Single Page Applications (SPAs) where commercial scraping tools often fail.
2. **Robust Multi-Rotation OCR Pipeline (`Tess4J`):**
   * Automatically processes uploaded newspaper clippings by executing 4-way image rotation, evaluating readability scores to select the optimal orientation, and filtering out margin noise lines before passing text to the LLM.
3. **4-Stage Validation & Retry Engine:**
   * Programmatically validates LLM-generated JSON responses in Java before persistence.
   * Enforces strict schema rules: exact option counts, valid answer indices, Jaccard word-overlap checks to flag near-duplicate distractors, length-bias checks, and minimum explanation lengths.
   * Implements automated feedback loops—retrying up to 3 times by feeding validation errors directly back into the prompt context.
4. **Normalized 3NF Relational Architecture:**
   * Designed with a cascading relational schema (`Passage` ➔ `Question` ➔ `Option` and `Attempt` ➔ `Response`) ensuring clean data lifecycle management and fast analytical queries.

---

## System Architecture & Data Flow

```text
[ Input: Text / Image / URL / Daily RC ]
                 │
                 ▼
          [ QuizController ]
                 ├──► Image ➔ OcrService (4x rotation, noise filter)
                 └──► URL   ➔ ArticleFetcherService (selector density scoring)
                 ▼
        [ GeminiApiService ]
                 ├──► PromptFactory (Exam-specific taxonomy & difficulty)
                 ├──► Google Gemini API Call
                 └──► QuizValidator (Schema checks & up to 3x automated retries)
                 ▼
   [ Relational Persistence (MySQL 3NF) ]
                 ▼
         [ JSON Response ➔ Frontend ]
```

---

## Tech Stack

* **Backend Core:** Java 17, Spring Boot 3, Spring Data JPA, Hibernate
* **Database:** MySQL 8.0 (Normalized 3NF Schema)
* **AI / LLM:** Google Gemini API (`gemini-3.5-flash-lite`)
* **OCR Processing:** Tesseract OCR via Tess4J wrapper
* **Web Extraction:** Jsoup HTML Parser
* **Frontend:** Vanilla HTML5, CSS3, JavaScript (Server-side static assets)

---

## Database Schema Model

* `Passage (1) ➔ (N) Question (1) ➔ (N) Option` (Cascading relational integrity)
* `Attempt (1) ➔ (N) Response` (Tracks anonymous session performance and latency)

---

## Getting Started & Local Installation

### Prerequisites
* Java 17 Development Kit
* Maven (or use the included `./mvnw` wrapper)
* MySQL 8.0 running locally
* Tesseract OCR (64-bit for Windows) installed at `C:\Program Files\Tesseract-OCR` with English language data (`eng.traineddata`).
* A Google Gemini API Key from [Google AI Studio](https://aistudio.google.com/).

### 1. Database Setup
Create the local MySQL database:

```sql
CREATE DATABASE rc_quiz_db;
```

### 2. Configure Environment Properties
Update or create your `src/main/resources/application.properties`:

```properties
gemini.api.key=${GEMINI_API_KEY}
gemini.api.url=[https://generativelanguage.googleapis.com/v1beta/models/gemini-3.5-flash-lite:generateContent](https://generativelanguage.googleapis.com/v1beta/models/gemini-3.5-flash-lite:generateContent)
admin.secret.key=${ADMIN_SECRET_KEY}

spring.datasource.url=jdbc:mysql://localhost:3306/rc_quiz_db?useSSL=false&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=YOUR_MYSQL_PASSWORD

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

*(Note: Never commit hardcoded API keys. Set them as environment variables.)*

### 3. Set Environment Variables (IntelliJ / Terminal)
* `GEMINI_API_KEY`: Your Google AI Studio API key
* `ADMIN_SECRET_KEY`: A secure random string for admin endpoints

### 4. Run the Application
Execute via Maven wrapper:

```bash
./mvnw spring-boot:run
```

Open your browser and navigate to `http://localhost:8080/`.

---

## API Reference Overview

* `POST /api/quiz/generate?examType=CAT` — Generates quiz set from raw article text body
* `POST /api/quiz/generate-from-image?examType=CAT` — Multipart form upload (`image`) processed via OCR
* `POST /api/quiz/generate-from-url?examType=CAT` — Extracts and generates quiz set from article URL
* `GET /api/quiz/daily` — Fetches today's admin-curated Reading Comprehension
* `POST /api/quiz/submit` — Submits user answers for server-side scoring
* `GET /api/quiz/stats/mine` — Retrieves accuracy breakdown by question type for the session

*Supported `examType` values:* `CAT`, `GMAT`, `SBI_PO`, `IBPS_PO`, `UPSC` (defaults to a balanced prompt if unspecified).

---

## Known Limitations & Production Roadmap

* **Anonymous Sessions:** User tracking relies entirely on browser `localStorage` session IDs without database user accounts.
* **Bot-Protected URLs:** Highly restricted domains (e.g., sites enforcing strict Cloudflare challenges or HTTP 429 rate limits like Aeon) will trigger a clean fallback prompt requesting manual text pasting.
* **Roadmap:** Implementation of strict per-session rate limiting, containerization (Docker), and cloud deployment on Render/Railway.

---

## License

Distributed under the MIT License. See `LICENSE` for more information.

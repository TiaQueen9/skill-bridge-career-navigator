# Design Documentation: Skill-Bridge Career Navigator

## Table of Contents
1. [Tech Stack & Why](#tech-stack--why)
2. [Project Architecture](#project-architecture)
3. [Module Breakdown & Functions](#module-breakdown--functions)
4. [Data Layer](#data-layer)
5. [AI Integration](#ai-integration)
6. [UI Design](#ui-design)
7. [Testing Strategy](#testing-strategy)
8. [Future Enhancements](#future-enhancements)

---

## Tech Stack & Why

| Technology | Role | Why We Chose It |
|---|---|---|
| **Python 3.12** | Core language | Fast prototyping, rich ecosystem for AI/data, widely understood |
| **Streamlit** | Web UI framework | Zero HTML/CSS/JS needed. Built-in session state, reactive widgets, forms — ideal for a 4-6 hour prototype. One file = full web app |
| **Groq API (Llama 3.3 70B)** | AI engine | Free tier (30 req/min), OpenAI-compatible SDK, fast inference (~1s responses), no credit card required |
| **python-dotenv** | Config management | Loads `.env` file for API keys — keeps secrets out of code |
| **pytest** | Unit testing | Industry standard, minimal boilerplate, great fixture system |
| **Hypothesis** | Property-based testing | Generates random inputs to find edge cases humans miss (available but optional for MVP) |
| **JSON** | Data storage | Simple, human-readable, no database setup needed for a prototype |

**Why Streamlit over Flask?**
Flask would require writing HTML templates, CSS, JavaScript, and route handlers. Streamlit gives us an interactive web app with just Python — forms, buttons, metrics, columns, expanders — all built-in. For a timed prototype, this saves 2+ hours.

**Why Groq over OpenAI/Gemini?**
Groq offers a genuinely free tier with no credit card. The SDK is OpenAI-compatible (easy to swap later). Llama 3.3 70B is a strong model for categorization tasks. Response times are under 1 second.

**Why no database?**
Streamlit's `st.session_state` gives us in-memory persistence for the session. For a prototype demo, this is sufficient. A real product would use PostgreSQL or similar.

---

## Project Architecture

```
skill-bridge/
├── app.py                      # Streamlit UI — the entry point
├── models.py                   # Data models (7 dataclasses)
├── profile_manager.py          # Profile CRUD + validation
├── resume_parser.py            # Skill extraction from text
├── profile_printer.py          # Profile → readable text
├── job_catalog.py              # Load, validate, search jobs
├── gap_analyzer.py             # Skill gap comparison engine
├── ai_engine.py                # Groq AI + rule-based fallback
├── roadmap_generator.py        # Learning plan builder
├── data/
│   ├── jobs.json               # 10 synthetic job postings
│   ├── skill_taxonomy.json     # 64 skill entries
│   └── learning_resources.json # 97 learning resources
├── tests/
│   ├── conftest.py             # Shared test fixtures
│   └── test_gap_analysis.py    # Unit tests (happy path + edge case)
├── .env.example                # API key placeholder
├── .gitignore                  # Keeps .env and caches out of git
├── requirements.txt            # Python dependencies
└── README.md                   # Project overview and instructions
```

**Data flow:**
```
User Input → Profile Manager → Session State
                                    ↓
Job Catalog ← search/filter ← User Selection
                                    ↓
Gap Analyzer ← (profile + job) → GapResult
                                    ↓
AI Engine (Groq or Fallback) → CategorizationResult
                                    ↓
Roadmap Generator → Phased Learning Plan
                                    ↓
User marks completed → Recalculate match %
```

---

## Module Breakdown & Functions

### 1. `models.py` — Data Models

Seven Python dataclasses that define the shape of all data in the system:

| Dataclass | Purpose | Key Fields |
|---|---|---|
| `UserProfile` | User's career profile | name, skills (list), experience_years, education, target_role |
| `JobPosting` | A job from the catalog | title, description, required_skills, preferred_skills, experience_level |
| `GapResult` | Output of gap analysis | matched_required, missing_required, matched_preferred, missing_preferred, match_percentage |
| `CategorizationResult` | AI categorization output | groups (dict of category → skills), summary (text), is_fallback (bool) |
| `LearningResource` | A single course/project/cert | name, skill, resource_type, estimated_hours, url, completed |
| `RoadmapPhase` | One time period in the roadmap | label ("Month 1-2"), resources (list) |
| `Roadmap` | The full learning plan | phases (3 RoadmapPhases) |

### 2. `profile_manager.py` — Profile CRUD & Validation

| Function | What It Does |
|---|---|
| `create_profile(name, skills, experience_years, education, target_role)` | Validates all inputs, deduplicates skills, returns `(UserProfile, notification)`. Raises `ValueError` with specific messages for: missing name/target_role, empty skills, skills > 30, skill name > 100 chars |
| `update_profile(profile, added_skills, removed_skills)` | Returns a new UserProfile with skills added/removed. Validates the resulting list |
| `save_profile(profile)` | Stores profile in `st.session_state["user_profile"]` |
| `load_profile()` | Retrieves profile from session state, or `None` |
| `_deduplicate_skills(skills)` | Internal helper — removes case-insensitive duplicates, returns `(deduped_list, had_duplicates)` |

**Validation rules enforced:**
- Name and target_role are required (non-empty)
- Skills list: 1–30 items
- Each skill name: ≤ 100 characters
- Duplicates auto-removed with user notification

### 3. `resume_parser.py` — Skill Extraction

| Function | What It Does |
|---|---|
| `load_taxonomy(path)` | Loads `skill_taxonomy.json` — a flat array of 64 skill strings |
| `parse_resume(text, taxonomy)` | Scans text for skills using case-insensitive word-boundary regex. Multi-word skills (e.g., "Machine Learning") matched as phrases. Returns list of found skills, empty list if none |

**How matching works:**
- Taxonomy sorted longest-first (so "Machine Learning" matches before "Machine")
- Each skill becomes a regex pattern: `\bMachine Learning\b` with `re.IGNORECASE`
- Deduplicates results (case-insensitive)

### 4. `profile_printer.py` — Profile Serialization

| Function | What It Does |
|---|---|
| `format_profile(profile)` | Converts a UserProfile to readable text: "Name: ...\nSkills: Python, SQL, ...\nExperience: 2 years\n..." |

### 5. `job_catalog.py` — Job Data Management

| Function | What It Does |
|---|---|
| `load_jobs(path)` | Loads `jobs.json`, validates each entry has all required fields. Skips malformed entries with a logged warning. Raises `FileNotFoundError` if file missing |
| `search_jobs(jobs, keyword, skill)` | Filters jobs by title keyword OR required skill (case-insensitive). Returns all jobs if both filters empty |

### 6. `gap_analyzer.py` — Skill Gap Comparison

| Function | What It Does |
|---|---|
| `analyze_gap(profile, job)` | Compares user skills against job requirements (case-insensitive). Partitions into matched/missing for required and preferred. Calculates `match_percentage = round(matched_required / total_required × 100)`. Returns 100% if job has 0 required skills |

### 7. `ai_engine.py` — AI Categorization with Fallback

| Class/Function | What It Does |
|---|---|
| `SkillCategorizerInterface` (ABC) | Abstract base — defines `categorize(missing_skills, matched_skills) → CategorizationResult` |
| `GroqCategorizer` | Calls Groq API (Llama 3.3 70B) with a structured prompt. 5-second timeout. Falls back on any error |
| `FallbackCategorizer` | Rule-based: maps skills to categories using keyword dictionary. Generates 2-4 sentence summary |
| `get_categorizer()` | Factory: returns `GroqCategorizer` if API key set, otherwise `FallbackCategorizer` |

**Fallback triggers:** API key missing, timeout >5s, any API error

### 8. `roadmap_generator.py` — Learning Plan Builder

| Function | What It Does |
|---|---|
| `generate_roadmap(gap, resources)` | Maps missing skills to resources across 3 phases. Required skills in earlier phases, preferred in later |
| `mark_completed(roadmap, resource_name)` | Returns new Roadmap with named resource marked completed |
| `recalculate_match(profile, job, roadmap)` | Recalculates match % treating completed resources' skills as acquired |

---

## Data Layer

- **skill_taxonomy.json** (64 entries) — programming languages, frameworks, cloud, databases, DevOps, data science, security, soft skills
- **jobs.json** (10 postings) — Backend Dev, Frontend Dev, Full Stack, Data Scientist, DevOps, Cloud Architect, ML Engineer, Cybersecurity Analyst, Mobile Dev, Project Manager
- **learning_resources.json** (97 resources) — courses, projects, certifications covering all job-required skills

All data is synthetic — no real personal information.

---

## AI Integration

1. User runs gap analysis → lists of missing/matched skills
2. `get_categorizer()` checks for `GROQ_API_KEY`
3. If available → Groq API categorizes skills and generates summary
4. If unavailable → fallback uses keyword mapping + template summary
5. UI shows clear label when fallback is active

---

## UI Design

Single-page Streamlit app with 3 progressive sections:

1. **Profile Creation** — resume paste (optional), profile form with skill multiselect, current profile display
2. **Job Catalog & Gap Analysis** — search/filter, job selector, gap dashboard with match %, AI categorization
3. **Learning Roadmap & Updates** — phased plan with "Mark Done" buttons, profile update with re-analysis

---

## Testing Strategy

| Test | What It Verifies |
|---|---|
| `test_happy_path_gap_analysis` | Profile with some skills → correct missing skills, correct match % |
| `test_edge_case_zero_skills` | Empty profile → all required skills missing, 0% match |
| `test_full_match` | All required skills present → 100% match |

---

## Future Enhancements

1. Real job board API integration (LinkedIn/Indeed)
2. Resume PDF upload with OCR
3. Persistent storage (PostgreSQL)
4. Mock interview generator
5. Visual charts for gap visualization
6. Multi-role comparison

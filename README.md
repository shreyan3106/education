# Study Companion

An AI study companion for **Multimodal AI Hackathon 2026, Track D (Personalized Tutoring and Adaptive Learning)**.
Upload course material and ask questions: every answer is built only from your material, cites the exact page, and is
checked by a second model from a different model family. Questions the material does not cover are declined.

## What works now

| Area | Status |
|---|---|
| PDF reading with heading detection (code first, language model assists) | working |
| Topic, key concept and prerequisite tags for every unit | working |
| Automatic ingestion on upload (read, clean, embed, save, tag) | working |
| Hybrid search (meaning + keywords) with a model reranker | working |
| Grounded tutor chat: citations, "show page", follow-up questions, refusal of off-material questions | working |
| Writer model + checker model from a different family (verified / corrected / unverified badge) | working |
| Model ladder with fallbacks, usage counter, cache | working |
| PowerPoint (.pptx) ingestion: every slide is a unit with title, text, speaker notes and described pictures; a cited slide opens as a card | built, first run in the real app pending |
| YouTube ingestion (up to 60 minutes): captions first, Whisper on Groq if there are none; a cited moment opens the video at that second | built, first run in the real app pending |
| Courses like projects: create a course (name, description, tutor instructions), add PDFs, slides and videos INTO it, move material between courses, rename or delete a course; every course has its own tutor chat, quizzes, dashboard and progress; the Course tab is the course's home | built, first run in the real app pending |
| Tutor: short answers, coach mode for questions about your own progress (from quiz results, no sources), quiz notes under answers | built, first run in the real app pending |
| Model check tab: benchmark that measures every model on real tasks and saves a best-first order | built, first run in the real app pending |
| Quizzes (topic quiz of 7) and mock exams (30 questions, 5 topics): MCQ, short answer, numerical; answer keys prepared by a second model family and cross-checked by code; no repeated questions; instant feedback; report with weak topics and likely misconceptions | built, first run with real models pending |
| Learner model (BKT + IRT-style + spaced repetition), diagnostic quiz and intake for new students, Dashboard tab, tutor and quizzes adapted to the student | built, first run in the real app pending |
| Time rules for every model call: 60 s to START answering (thinking counts), no total limit while it answers, 60 s of silence mid-reply = hand-over; a slow model rests 10 min (saved across restarts) | built, first run in the real app pending |
| Hand-over mid-reply: the fastest remaining model takes over at once, with the same conversation; tutor answers are continued from the partial text, strict-format jobs start the same task again | built, first run in the real app pending |
| Extra models: inclusionAI Ling 3.1 Flash, Apodex 1.1 Mini, Nemotron 3 Super (second route via OpenRouter); rerank fallback with NVIDIA Llama Nemotron Rerank VL 1B (OpenRouter rerank endpoint) | built, first run in the real app pending |
| Mastery = quiz, mock exam and diagnostic answers only (each question once); few answers stay near the starting level, the table shows evidence ("4 of 5 right"), a likely range and review dates in plain words; weak topics, mistakes and thin evidence come before untested topics in "Study next"; checked on simulated students (mean error, range coverage, correlation) | built and tested with simulated students |
| Tutor keeps progress questions ("why is this topic due?") with the coach, also as follow-ups, and never sends them to general knowledge | built, first run in the real app pending |
| Title pages, greetings and tiny pages are not topics | built, first run in the real app pending |
| Evaluation (RAGAS, simulated students) | simulated students for the learner model: tested (tests/test_learner.py, test 17); RAGAS not built yet |
| Scanned PDF pages (read by a vision model) and figure / diagram description (vision model, 30 pages at a time) | built, first run in the real app pending |
| Textbooks up to 450 pages | built, first run on a full textbook pending |

## Run it

1. Python 3.11 or 3.12. In the project folder:
   ```
   python -m venv venv
   venv\Scripts\activate            (Windows)   or   source venv/bin/activate   (Mac / Linux)
   pip install -r requirements.txt
   python download_embedding_model.py
   ```
2. Copy `.env.example` to `.env` and paste **at least one** free API key:
   - Groq: console.groq.com (free tier, rate limited)
   - NVIDIA: build.nvidia.com (free endpoints; a phone check may apply in some countries)
   - OpenRouter: openrouter.ai (a few free models, small daily cap)
   - Hugging Face: huggingface.co/settings/tokens (a "Read" token, put in `.env` as `HF_TOKEN`): authenticates model downloads and gives a backup for models that read pictures (free monthly credits are small)
   Never share, upload or submit your `.env` file; `.gitignore` already excludes it, together with `data/` and `venv/`.
3. `streamlit run app.py`, create a course in the sidebar, add a PDF, slides or a YouTube link in the **Course** tab, then open the **Tutor chat** tab.

## How it works

```
upload PDF -> read text, fonts, positions (PyMuPDF) -> headings by learning the document's own style
           -> LLM removes false headings -> units (heading + text) -> chunks of ~180 words (page kept)
           -> local embeddings -> SQLite (units, chunks, vectors, topic/concept tags)

question -> (follow-up? model rewrites it as a standalone question)
         -> hybrid search: embeddings (FAISS) + keywords (BM25), merged by rank
         -> skip worked examples / exercises unless asked about -> reranker model picks the best 4 passages
         -> too far from everything saved? decline (no writer call)
         -> writer model answers only from the numbered passages with [n] citations
         -> checker model (different family) verifies or corrects -> answer + sources + badge

upload -> headings, chunks, embeddings, tags -> OUTLINE HELPER sorts every heading (title / section /
          subsection / example / exercise / summary / other) and groups sections into topics -> saved table

quiz topic -> passages (search, or the chunks tagged with a chosen topic; least-used passages first)
          -> setter model writes the questions only (never the answers)
          -> questions too similar to earlier ones are dropped (saved history, vectors)
          -> checker model (different family) writes the answer key and must copy the proving sentence
          -> code: the sentence must be in the passage; numerical answer = result of our own calculator
          -> saved with tags (topic, concept, difficulty, source page) -> answered with instant feedback
          -> Submit: score, weak topics, likely misconceptions, pages to revise
```

Every model call goes through `src/llm.py`: a ladder of models per job (answer, extract, check, rewrite, rerank) over
three providers (NVIDIA, Groq, OpenRouter). It skips models without a key or over their free limit (counted in SQLite),
moves on after errors, timeouts or cut-off replies, and caches results.

## Files

| File | Job |
|---|---|
| `app.py` | the Streamlit app: tabs Tutor chat, Knowledge base, Quiz, Dashboard, Model check |
| `src/ingest_pdf.py` | reads a PDF, finds headings, builds units |
| `src/organize.py` | model help: clean the heading list, tag topics, concepts and prerequisites |
| `src/chunking.py` | splits units into search-sized chunks and keeps page numbers |
| `src/embed.py` | local embedding models (small and fast first) |
| `src/store.py` | SQLite tables, saved files, tags |
| `src/search.py` | hybrid search (FAISS or NumPy, rank-bm25 or built-in BM25, rank fusion) |
| `src/tutor.py` | the grounded tutor: rewrite, retrieve, rerank, write, check |
| `src/learner.py` | the learner model: Bayesian knowledge tracing (main), IRT-style ability and spaced repetition (supporting); recommendations, difficulty plan, tutor guidance |
| `src/learner_ui.py` | the Dashboard tab: intake, mastery per topic, study next, how the numbers are made |
| `src/outline.py` | the document outline helper: a model sorts every heading (role) and groups sections into topics; rules as fallback; used by the tutor and the quiz |
| `src/pipeline.py` | the one save path for PDFs, slide decks and videos: chunk, embed, save, tag, sort, assign to a course |
| `src/slides.py` | reads .pptx decks (slide = unit); the slide card |
| `src/video.py` | YouTube links: captions or Whisper, one-minute passages, five-minute titled units (locations are seconds) |
| `src/figures.py` | vision models: describe figures, read scanned pages |
| `src/library.py` | courses (create, rename, settings, move material, delete) and the per-course learner record |
| `src/viewer.py` | opens a cited source: PDF page, slide card, or the video at that second |
| `src/coach.py` | answers questions about the student's own progress from quiz results (no sources) |
| `src/fmt.py` | math formatting: cleaning and a model repair step |
| `src/benchmark.py` | measures every model on real tasks and proposes the best-first order |
| `src/quiz.py` | quizzes and mock exams: passages, setter model, repeat check, checker model, code checks, grading, report |
| `src/quiz_ui.py` | the Quiz tab |
| `src/llm.py` | model providers, ladders, limits, cache |
| `src/pages.py` | renders the cited page as a picture with the passage highlighted |
| `tests/` | tests that run with fake models (no internet or keys): `python tests/test_llm.py` etc. |

## Settings you can change without writing code

| Setting | File | Meaning |
|---|---|---|
| `TOP_K`, `CANDIDATES` | `src/tutor.py` | passages sent to the writer, passages the reranker chooses from |
| `REFUSE_BELOW`, `MIN_CHUNK_SIM` | `src/tutor.py` | how similar a passage must be before the tutor answers |
| `GROQ_MODELS`, `NVIDIA_MODELS`, `OPENROUTER_MODELS`, `LADDERS` | `src/llm.py` | which models exist and the order they are tried (best first; the Model check tab can save a measured order in `data/ladders.json`, which wins) |
| `MAX_PAGES` | `src/ingest_pdf.py` | longest PDF accepted (450) |
| `MAX_SECONDS` | `src/video.py` | longest video accepted (60 minutes) |
| `START_WAIT`, `SILENCE_LIMIT`, `SLOW_REST` | `src/llm.py` | seconds a model has to start answering (60), seconds of silence mid-reply before a hand-over (60), minutes a slow model rests (600 s). There is no total limit while a model is answering |
| `RESUME_JOBS`, `MIN_RESUME_CHARS` | `src/llm.py` | jobs whose partial text is continued by the next model after a break, and the shortest partial text worth continuing |
| `RERANK_MODEL` | `src/llm.py` | the rerank model used when no language model could rerank |
| `EVIDENCE_PRIOR`, `MIN_EVIDENCE`, `MIN_REVIEW_GAP` | `src/learner.py` | how many answers the starting level counts for, answers needed before a topic is called weak or strong, shortest time (days) before a review is due |
| `MAX_INSTRUCTIONS` | `src/library.py` | longest tutor instructions of a course that are used (characters) |
| `MIN_TOPIC_WORDS` | `src/outline.py` | a topic needs at least this many words of teaching text (title pages and greetings are left out) |
| `EMBED_MODELS` | `src/embed.py` | embedding model order (add files again after changing it) |
| `TARGET_WORDS` | `src/chunking.py` | chunk size |
| `LEVEL_PRIOR`, `LEARN_QUIZ`, `GUESS`, `SLIP`, `RETENTION_TARGET`, `DECAY_FLOOR` | `src/learner.py` | learner-model settings: starting estimate by level, learning rate, guess and slip chances, recall level at which a review is due, how much forgetting can lower mastery |
| `WINDOW` | `src/outline.py` | headings sent to the helper model per call |
| `TOPIC_QUIZ_SIZE`, `MOCK_SIZE`, `MOCK_TOPICS` | `src/quiz.py` | questions in a topic quiz, in a mock exam, topics a mock exam needs |
| `REPEAT_SIM`, `QUIZ_REFUSE_BELOW`, `NUMERIC_TOLERANCE`, `WEAK_BELOW` | `src/quiz.py` | how similar a question must be to count as a repeat, how close a typed topic must be to the material, accepted rounding error in numerical answers, score under which a topic is called weak |

## Notes

- Use public or openly licensed material only. Hosted free endpoints log requests (NVIDIA's free endpoint says so on its page; some OpenRouter free models may keep prompts to train their models); do not upload personal data.
- Parts of the code were written with the help of an AI assistant (Claude, by Anthropic), directed, run and tested by the team.
  Edit this line to match the hackathon's rules on disclosure.

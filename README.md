Effective C++ Advisor
An agentic code review tool for C++ that combines static analysis,
retrieval-augmented generation (RAG), and a model-agnostic LLM
layer to give grounded, principle-based explanations of common C++
design issues — const-correctness, RAII, virtual destructors, implicit
conversions, and more.
Instead of asking an LLM to "review this code" and hoping it notices
the right things, this project separates detection (deterministic
static analysis) from explanation (LLM generation grounded in a
curated knowledge base via RAG). Detection tells you what is wrong
and where, with certainty. RAG + the LLM tell you why it matters
and how to fix it, grounded in reference material instead of the
model's unverified recollection.
```
$ python -m cli.main review examples/sample_violations.cpp --provider local

examples/sample_violations.cpp: 5 finding(s)
------------------------------------------------------------
[item_07_virtual_destructors] examples/sample_violations.cpp:4
  class Shape {
  -> Shape declares a virtual `area()` method, marking it as a
     polymorphic base class, but its destructor isn't virtual. Deleting
     a derived object (e.g. a Circle) through a Shape* will only run
     ~Shape(), silently skipping the derived destructor. Fix: declare
     `virtual ~Shape() = default;`.
...
------------------------------------------------------------
```
Architecture
```
parse (static_analysis/) -> detect -> retrieve (rag/) -> generate (agent/)
```
Detection — pattern detectors scan C++ source for known issues
(raw pointer ownership, missing const, missing virtual destructor,
non-explicit single-arg constructors). Each detector maps 1:1 to a
`skill_id`.
Retrieval — the matching `SKILL.md` document (plus related
items) is fetched from a local vector store built from the
`skills/` knowledge base.
Generation — the LLM is prompted with the specific code snippet
and the retrieved reference material, and asked to produce a
grounded, specific explanation — not a generic lecture.
```
effective-cpp-advisor/
├── skills/effective_cpp/       # curated knowledge base (SKILL.md per principle)
├── rag/                        # ingestion, vector store, retrieval
│   └── embeddings/             # embedding provider interface (local/hosted)
├── static_analysis/
│   └── pattern_detectors/      # one detector per detectable principle
├── agent/
│   ├── llm_provider/           # LLM provider interface (Anthropic/OpenAI/local)
│   ├── prompts/                # versioned prompt templates (not hardcoded strings)
│   └── orchestrator.py         # ties detect -> retrieve -> generate together
├── cli/                        # `review` and `ingest` commands
├── examples/                   # sample file with intentional violations
└── tests/test_detectors/       # detector unit tests, no LLM required
```
Setup
```bash
pip install -r requirements.txt      # or a subset — see requirements.txt comments

# Build the RAG vector store from skills/effective_cpp/*/SKILL.md
python -m cli.main ingest

# Review a file — pick any provider
python -m cli.main review examples/sample_violations.cpp --provider local      # Ollama, fully offline
python -m cli.main review examples/sample_violations.cpp --provider anthropic  # needs ANTHROPIC_API_KEY
python -m cli.main review examples/sample_violations.cpp --provider openai     # needs OPENAI_API_KEY
```
Design decisions
Detection is deterministic, not LLM-driven. For a code review tool,
having the LLM decide what to look for is less reliable than
pattern-based static analysis, and it makes detection testable without
a live model call (`tests/test_detectors/` runs with zero API cost).
The LLM's job is explanation, not discovery.
RAG over fine-tuning. The knowledge base is ~20 curated `SKILL.md`
files, each independently editable, versioned, and diffable in a PR.
Fine-tuning a model on this content would be slower to iterate on,
opaque about what the model actually "knows," and tied to one provider.
RAG keeps the knowledge base swappable, inspectable, and provider-agnostic.
Provider abstraction is real, not cosmetic. `LLMProvider` and
`EmbeddingProvider` are the only interfaces the rest of the codebase
depends on. The `local` provider (Ollama) proves this: the full
pipeline runs with zero API keys and zero network calls.
Regex-based detectors, not a full AST parser, for this scaffold.
Each detector implements `PatternDetector.detect()` in
`static_analysis/detector_registry.py`. The natural production upgrade
is swapping the regex implementations for a `libclang`/`clang-tidy`
AST visitor — the interface doesn't change, only what's inside each
`detect()` method, which is the point of keeping detection behind an
interface in the first place.
Prompts are versioned markdown files, not inline strings. See
`agent/prompts/review_explainer.prompt.md` — this makes prompt changes
reviewable in diffs and separable from orchestration code.
Extending
New detectable principle: add a `skills/effective_cpp/item_XX/SKILL.md`
(use `_template/SKILL.md`), write a detector implementing
`PatternDetector`, register it in `detector_registry.py`, re-run `ingest`.
New LLM backend: implement `LLMProvider`, add a branch in
`get_provider()`.
New embedding backend: implement `EmbeddingProvider`, swap the
return of `get_default_provider()`.
Status
Scaffold / portfolio project. Detectors are intentionally simple
(regex-based) to keep the project buildable in days rather than weeks;
the architecture is designed so the detection layer can be swapped for
a real AST-based implementation without touching retrieval, generation,
or the CLI.

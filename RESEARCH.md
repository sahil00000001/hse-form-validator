# HSE Form Validator — Step 1: GitHub reuse check

**Date:** 2026-10-29 · **Status:** awaiting owner approval (no code written yet)

This report checks the planned stack for an HSE form validator before any code is written.
The validator takes a form (PDF/DOCX/XLSX) and checks it against a human-approved checklist of
requirements extracted from regulations. Every verdict must cite a verbatim clause quote.
The report answers three questions:

1. Is each planned library still maintained, safely licensed, and what exact API will we call?
2. Does anything on GitHub already do all or part of this job?
3. Keep, replace or add each component?

**Method.** Three parallel research passes were merged into this report:

- **Core stack:** Hermes, Docling, MCP and embeddings.
- **LLM layer:** LangExtract, PydanticAI, pydantic-evals, GEPA and the free tiers.
- **Alternatives:** a GitHub/PyPI search for existing tools.

Facts were read from repo source and docs at the latest release tag via the GitHub API. Release
dates and licences come from `gh api`. The most surprising claims were re-checked by hand.
Items that could not be verified are listed at the end.

---

## 1. Decision table

| Repo (latest release) | Decision | Why | Licence risk |
|---|---|---|---|
| [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) v0.22.5 (2026-10-23) | **Keep** | Very active. Provides the Telegram gateway, skills, memory, cron and MCP client. Native Windows is now Tier 1. Verified config keys: `mcp_servers`, `skills.write_approval` and `fallback_providers`. | MIT · none |
| [docling-project/docling](https://github.com/docling-project/docling) v2.144.0 (2026-10-28) | **Keep** | Parses PDF/DOCX/XLSX/HTML with page provenance. `extract_form_fields=True` turns AcroForm widgets into `FieldItem`s. Two gaps: no page numbers for DOCX, and filled-in field values are still an open issue ([#2677](https://github.com/docling-project/docling/issues/2677)). | MIT · none |
| [py-pdf/pypdf](https://github.com/py-pdf/pypdf) 6.21.4 (2026-10-28) | **Keep** (fallback) | Reads filled AcroForm values and checkbox/radio states, which Docling can't yet. Also a cheap text fallback. | BSD-3-Clause · none |
| [docling-project/docling-mcp](https://github.com/docling-project/docling-mcp) v3.4.0 (2026-10-27) | **Don't use**: import Docling in-process | Defaults to HTTP transport and *remote* conversion. Exposes no form-field option and no page provenance. Adds a subprocess for nothing we need. | MIT · none |
| [modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk) v2.3.0 (2026-10-02) | **Keep**, use the v2 API | `FastMCP` was renamed to `MCPServer` in v2, and the old import fails. Hermes itself pins `mcp==2.3.0`. | MIT · none |
| [pydantic/pydantic-ai](https://github.com/pydantic/pydantic-ai) v2.63.0 (2026-10-28) | **Add**: replaces the hand-rolled OpenAI-compatible client in the plan (§6) | Built in: `FallbackModel`, native Gemini/Groq/OpenRouter/Ollama models, `output_type=` Pydantic validation, retry transport honouring `Retry-After`. `TestModel`/`FunctionModel` mock every LLM call in tests. | MIT · none |
| pydantic-evals (same repo) | **Add**: the `hsev eval` harness and golden set | `Dataset`/`Case`/custom `Evaluator`. YAML save/load makes golden tests reviewable text files in git. | MIT · none |
| [google/langextract](https://github.com/google/langextract) v1.7.1 (2026-10-08) | **Add**: the regulation → requirement extraction step | Grounding is built in: every extraction gets `char_interval` + `alignment_status`, or `None` if not found → maps directly to `NEEDS_REVIEW`. Handles long-document chunking and multi-pass. Trade-off: it has its own LLM client, so extraction fallback is ours to wire. | Apache-2.0 · none (keep NOTICE if redistributed) |
| [gepa-ai/gepa](https://github.com/gepa-ai/gepa) v0.1.4 (2026-07-15, commits to 2026-10-28) | **Add, later** (M4+): judge-prompt optimisation | `optimize()` with a custom adapter can improve the judge prompt from expert-corrected cases. The core has zero dependencies if we pass callables, and no DSPy is needed. Output is a *candidate* prompt that a human must approve before activation. | MIT · none |
| [huggingface/sentence-transformers](https://github.com/huggingface/sentence-transformers) v6.2.0 + [BAAI/bge-m3](https://huggingface.co/BAAI/bge-m3) | **Keep** | Local multilingual embeddings for field matching. bge-m3 ships `.bin` weights only (2.27 GB), so it needs **torch ≥ 2.6**. | Apache-2.0 / MIT · none |
| [ollama/ollama](https://github.com/ollama/ollama) v0.40.12 (2026-10-27) | **Keep** | The `local_only` route. `qwen3.5:9b` exists (6.6 GB q4). Has a native Windows installer and JSON-schema output on `/v1`. | MIT · none |
| Typer 0.27.3 · Pydantic 2.14.1 · pytest 9.2.0 | **Keep** | Standard and active. | MIT · none |
| [legislation/legislation-mcp-ts](https://github.com/legislation/legislation-mcp-ts) (The National Archives, official) | **Optional add** (UK only) | legislation.gov.uk with section-level fragments, extent and point-in-time versions. Useful for fetching exact UK clauses to ingest. TypeScript, so it needs Node. | OGL v3.0 · low (attribution) |
| [paulieb89/uk-legal-mcp](https://github.com/paulieb89/uk-legal-mcp) | **Optional alt** to the above | Python MCP for legislation.gov.uk plus case law. Single maintainer. | MIT · low |
| [geofila/ARCCS](https://github.com/geofila/ARCCS) | **Borrow ideas only** | The closest analogue found: regulation → atomic requirements → per-requirement compliance label with evidence and a `HUMAN_REQUIRED` class. Targets GDPR vs terms of service, not forms. | **High: no licence.** Do not copy code. |
| [anthropics/claude-for-legal](https://github.com/anthropics/claude-for-legal) (`reg-gap-analysis`) | **Borrow ideas only** | Adds a "does this rule apply here?" step before the gap diff, which suits HSE thresholds (e.g. 5+ employees). | Apache-2.0 · none |
| [i-dot-ai/lex](https://github.com/i-dot-ai/lex) | **Ignore** (ideas only) | UK legislation semantic search. Self-described experimental; needs Azure OpenAI + Qdrant. | MIT · none |
| [stark256-spec/regulations-mcp](https://github.com/stark256-spec/regulations-mcp) | **Ignore** | US federal only (Federal Register, eCFR, Regulations.gov); no UK or India coverage. 3 commits, all on 2026-06-01; alpha. | **Medium–high:** claims MIT but has no LICENSE file |
| PyMuPDF, LynnHaDo/Checkbox-Detection | **Avoid** | Would solve form/checkbox parsing but are AGPL. | **High: AGPL** |

**Bottom line:** no existing repo does the end-to-end job (regulation → approved checklist →
form verdicts with evidence → learning from corrections), for HSE or anything else. Build it on
the stack above.

---

## 2. Exact APIs we will call (verified at the latest release)

### LangExtract: requirement extraction with grounding
```python
import langextract as lx
from langextract.factory import ModelConfig

result = lx.extract(
    text_or_documents=clause_text,
    prompt_description=EXTRACT_PROMPT,
    examples=[lx.data.ExampleData(...)],      # few-shot, synthetic only
    model_id="gemini-2.5-flash",              # set explicitly: default is "gemini-3.5-flash"
    extraction_passes=2, max_char_buffer=1000,
)
for e in result.extractions:
    e.char_interval      # CharInterval(start_pos, end_pos) or None -> NEEDS_REVIEW
    e.alignment_status   # MATCH_EXACT | MATCH_GREATER | MATCH_LESSER | MATCH_FUZZY
    e.attributes         # dict[str, str | list[str]] e.g. severity, frequency

# Groq / OpenRouter (OpenAI-compatible) need an explicit config:
config = ModelConfig(model_id="openai/gpt-oss-120b", provider="openai",
                     provider_kwargs={"api_key": ..., "base_url": "https://api.groq.com/openai/v1"})
```

**Keys:**
- `GEMINI_API_KEY`, falling back to `LANGEXTRACT_API_KEY`.
- `OLLAMA_BASE_URL` for Ollama.

**Watch out:**
- A model ID starting with `gpt-oss` auto-routes to Ollama. Groq must use `provider="openai"`.
- `lx.extract(..., model=...)` accepts a model instance, so tests can inject a fake.

**Install:** `langextract[openai]`.

### PydanticAI v2: judge, fallback chain, mocks
```python
from pydantic_ai import Agent, ModelSettings
from pydantic_ai.models.fallback import FallbackModel
from pydantic_ai.models.google import GoogleModel        # GOOGLE_API_KEY (or GEMINI_API_KEY)
from pydantic_ai.models.groq import GroqModel            # GROQ_API_KEY
from pydantic_ai.models.openrouter import OpenRouterModel  # OPENROUTER_API_KEY
from pydantic_ai.models.ollama import OllamaModel        # OllamaProvider(base_url="http://localhost:11434/v1")

model = FallbackModel(GoogleModel("gemini-2.5-flash"), GroqModel("openai/gpt-oss-120b"), ...)
judge = Agent(model, output_type=JudgeBatch,
              model_settings=ModelSettings(temperature=0.0, max_retries=0))
out = judge.run_sync(prompt).output                      # validated Pydantic object
```

**`local_only` mode:**
- It is just `Agent(OllamaModel(...))`, with no cloud model in the chain.
- The router refuses to build a cloud chain when the flag is set.

**Retries:**
- `pydantic_ai.retries.AsyncHTTPX2TenacityTransport` + `wait_retry_after(...)`. Needs the `[retries]` extra.
- Groq's SDK still needs the legacy `AsyncTenacityTransport`. It is deprecated and removed in v3.

**Tests:**
```python
from pydantic_ai import models
from pydantic_ai.models.test import TestModel
from pydantic_ai.models.function import FunctionModel
models.ALLOW_MODEL_REQUESTS = False
with judge.override(model=FunctionModel(fake_judge)): ...
```

**Install:** `pydantic-ai-slim[google,groq,openai,openrouter,retries,evals]`. Note:
- The full `pydantic-ai` bundle omits Groq.
- Ollama uses the `openai` extra.

### pydantic-evals: `hsev eval`
```python
from pydantic_evals import Case, Dataset
from pydantic_evals.evaluators import Evaluator, EvaluatorContext

class CriticalRecall(Evaluator[FormInput, Report]):
    def evaluate(self, ctx: EvaluatorContext[FormInput, Report]) -> float: ...

ds = Dataset(name="golden", cases=[...], evaluators=[CriticalRecall()])
report = ds.evaluate_sync(validate_form)          # report.cases, report.averages()
ds.to_file("eval/golden/golden.yaml", custom_evaluator_types=[CriticalRecall])
```

### GEPA: later, human-gated judge-prompt optimisation
```python
import gepa
result = gepa.optimize(seed_candidate={"judge_prompt": CURRENT},
                       trainset=..., valset=..., adapter=MyJudgeAdapter(),
                       reflection_lm=callable_or_litellm_string, max_metric_calls=150)
# result -> staged as a candidate prompt; never activated without expert approval
```

**Adapter:** a custom adapter implements `GEPAAdapter.evaluate()` and
`make_reflective_dataset()`. Pydantic published a reference pattern (`EvalsGEPAAdapter` using
`agent.override(instructions=...)` + pydantic-evals).

**Cost:** roughly 150 task calls + 10–30 reflection calls per run. This is inferred from the
code, not documented.

### Docling: parsing
```python
from docling.document_converter import DocumentConverter, PdfFormatOption
from docling.datamodel.base_models import InputFormat
from docling.datamodel.pipeline_options import PdfPipelineOptions

opts = PdfPipelineOptions(do_ocr=False, extract_form_fields=True)   # OCR only for scans
conv = DocumentConverter(format_options={InputFormat.PDF: PdfFormatOption(pipeline_options=opts)})
doc = conv.convert(path).document
for item, level in doc.iterate_items():
    item.prov[0].page_no if item.prov else None   # DOCX: no page provenance
```

**Notes:**
- `docling` is now a meta-package for `docling-slim[standard]`, which pulls in torch.
- Pin a CPU torch index with uv to avoid CUDA wheels.

### MCP Python SDK v2: `hsev mcp`
```python
from mcp.server import MCPServer          # NOT mcp.server.fastmcp (removed in v2)
mcp = MCPServer("hsev")

@mcp.tool()
def validate_form(path: str, ruleset: str) -> dict: ...

mcp.run()                                  # stdio by default; log to stderr only
```

### Hermes Agent: config (`config.yaml`)
```yaml
mcp_servers:
  hsev:
    command: "<venv>/Scripts/hsev.exe"     # Windows; Linux: <venv>/bin/hsev
    args: ["mcp"]
    env: {HSEV_HOME: "<project dir>"}      # stdio servers get only this env + a safe baseline
skills:
  write_approval: true                     # staged in pending/skills; /skills approve <id>
fallback_providers:
  - {provider: openrouter, model: "<free-model-id>:free"}
  - {provider: custom, model: "qwen3.5:9b", base_url: "http://localhost:11434/v1", key_env: "..."}
```

**Telegram:**
- Env vars: `TELEGRAM_BOT_TOKEN` and `TELEGRAM_ALLOWED_USERS`.
- Incoming documents up to 20 MB are saved locally, and the path is injected into the prompt.
- The agent sends a file back by emitting `MEDIA:/abs/path`. pdf/docx/xlsx are supported.

**Cron:**
- In chat: `/cron add "every 24h" "..." --skill hse-form-validator`.
- From the CLI: `hermes cron create "0 2 * * *" "..."`.

**Other commands:** `/reload-mcp` and the `skill_manage` tool both exist.

---

## 3. Corrections to the current plan (README)

| Plan says | Verified reality | Impact |
|---|---|---|
| §11 install: `curl … install.sh \| bash` | Linux/macOS/WSL2 only. Native Windows uses `iex (irm https://hermes-agent.nousresearch.com/install.ps1)` or MSIX. pip/uv installs are **unsupported**. | Pick the host first (see Q4) |
| §11 `~/.hermes/.env`, `~/.hermes/config.yaml` | On native Windows: `%LOCALAPPDATA%\hermes\` | Config snippet paths |
| §6 "one OpenAI-compatible client for all" | PydanticAI `FallbackModel` does this with native providers. LangExtract has its own client. | `llm.py` becomes a thin router over PydanticAI |
| §5 Docling "PDF/DOCX/XLSX… page refs" | DOCX items have **no page numbers**; XLSX sheets count as pages | Citations for DOCX use clause/heading + paragraph index, not page |
| §6 Ollama `qwen3.5:9b` | Exists. Ollama defaults to a **4k context** under 24 GB VRAM. Hermes needs ≥ 64K. | Set `OLLAMA_CONTEXT_LENGTH=64000` |
| §6 Gemini free models | Free per pricing page (2026-10-27): `gemini-2.5-pro`, `gemini-2.5-flash(-lite)`, `gemini-3-flash-preview`, `gemini-3.6-flash`, `gemini-3.8-flash`, `gemini-3.1/3.5-flash-lite`. Free RPM/RPD only visible in AI Studio. | Put model IDs in `config/llm.yaml`, not code |
| §6 Groq `gpt-oss-120b` 30 RPM / 1K RPD / 8K TPM / 200K TPD | Still listed with those numbers, but the page's Free vs Developer tab is ambiguous | Treat them as an upper bound; the client-side limiter is configurable |
| §6 OpenRouter `:free` 20 RPM, 50/day | Confirmed: 50/day below 10 credits, 1,000/day at ≥ 10 credits | No change |

---

## 4. Gaps nothing on GitHub covers (we build these)

- **The whole pipeline.** Requirement checklist → form verdicts with evidence → expert-corrected
  learning loop. ARCCS is the only near match, and it has no licence.
- **UK HSE guidance.** ACOPs and guidance (L74, HSG245, INDG…) are **not** on legislation.gov.uk
  and no MCP covers them. We ingest those PDFs ourselves with Docling.
- **India.** No usable regulation source or MCP was found. Unverified: the Factories Act 1948 is
  being replaced by the OSH Code 2020 with state-level rules. Which one the India checklist
  should cite needs confirming.
- **Filled form values.** pypdf handles PDF AcroForms. DOCX content controls need direct XML
  reading. Scanned checkboxes need OCR, and the only good open tools are AGPL.

---

## 5. Open decisions for the owner

1. **Stack additions.** Approve adding PydanticAI, pydantic-evals, LangExtract and GEPA (GEPA
   later), and the optional UK legislation MCP? None are in the README §5 table yet.
2. **Milestones.** The README §12 build plan stops at **M6**. The build brief says **M0 → M7**.
   What is M7, or is there a newer README?
3. **Scope.** The source list (`uk_hse_sources.csv`) is UK/Great Britain, while the README
   examples are India (`IN/factory`). UK first, India first, or both? (US/OSHA appears out of
   scope.)
4. **Hermes host for the pilot.** Native Windows (`install.ps1`), WSL2, or the Oracle Always
   Free VM?

---

## 6. Not verified

- Free-tier per-model RPM/RPD for Gemini (only visible in AI Studio). Whether LangExtract's
  default `gemini-3.5-flash` is on the free tier.
- Groq's Free-plan numbers, separately from the Developer plan.
- Whether `GroqModel` accepts the new httpx2 retry client.
- Actually loading bge-m3 on this machine, and its RAM use (estimated 3–4 GB).
- Whether PyPI torch wheels on Windows are CPU-only by default.
- How reliably a small local model passes a Telegram file path into an MCP tool call.
- GEPA LLM-call counts (inferred from code).
- Reg2Req (arXiv 2607.04448), a requirement-extraction method with cross-reference handling:
  the licence of its code on Zenodo was not checked.

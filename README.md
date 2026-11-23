# Machine learning compulsory 2

An AI research assistant built with PydanticAI that searches academic papers via OpenAlex API, enforces publication year and citation count constraints deterministically, and returns verified, structured recommendations.



## LLM Choice & Setup

 For this assignment, I used Google AI Studio (Gemini API) using a FallbackModel chain of `google:gemini-3.5-flash-lite` and `google:gemini-3.1-flash-lite`. Both models support native tool calling and structured outputs in PydanticAI and are available on Google's free tier without a billing account, while being more available than the newere models. I initially tested `gemini-3.8-flash`, but it frequently returned 503 UNAVAILABLE high-demand errors. 


## Installation & Configuration

### 1. Clone the repository and install dependencies
```bash
pip install -r requirements.txt
```


### 2. Configure the API Key
Create a .env file in the root directory of the project and put a Google Api Key inside :

```env
GOOGLE_API_KEY=google_ai_studio_api_key
```

If neccesary, can also add a OpenAlex Api Key : 
```env
OPENALEX_API_KEY=openalex_api_key
```


### 3. Run the Agent & Evaluation
Open `assingment.ipynb` in VS Code or Jupyter Notebook and run the cells in order:


**Cell 1 (Main Agent Workflow):** Initializes the PydanticAI agent, registers the OpenAlex search tool, configures the deterministic output validator, and runs a sample research query.

**Cell 2 (Evaluation Suite):** Executes the 10-prompt test suite, prints the summary Markdown table, and exports detailed outputs to `eval_results.json`.


## Evaluation Method & Results

### Evaluation Methodology
Following standard AI agent evaluation practices, this project uses a hybrid evaluation:
* **Deterministic code graders** automatically verify hard constraints for every test case: publication year bounds, citation count thresholds, valid OpenAlex source links, and strict grounding against retrieved tool data (no hallucinated metadata).
* **Manual review** evaluates subjective quality that code alone cannot verify: whether the selected paper is truly relevant to the topic and whether the explanation is useful.
* The 10-prompt evaluation set covers broad topics, narrow topics, before/after/exact year constraints, minimum citation thresholds, impossible/anachronistic queries, and ambiguous requests.

### Test Results (8/10 Passed)

* **Test 1 — Broad + After Year + Min Citations:** `PASS`
  * **Paper:** *AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation* (2023) | **Citations:** 4,231 (OpenAlex)
  * **Checks:** Year: PASS | Citations: PASS | Source: PASS | No Hallucination: PASS
* **Test 2 — Broad + Before Year + High Citations:** `PASS` *(self-corrected after validator caught a citation mismatch)*
  * **Paper:** *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks* (2020) | **Citations:** 10,595 (OpenAlex)
  * **Checks:** Year: PASS | Citations: PASS | Source: PASS | No Hallucination: PASS
* **Test 3 — Exact Year + Min Citations:** `FAILED`
  * **Error:** `All models from FallbackModel failed (2 sub-exceptions)` (external Google API availability/rate-limit error before tool execution)
  * **Checks:** Year: FAILED | Citations: FAILED | Source: FAILED | No Hallucination: FAILED
* **Test 4 — Narrow Topic + After Year:** `PASS` *(deterministic checks passed; manual review notes partial topic drift—paper is a code LLM report rather than strictly multi-agent)*
  * **Paper:** *Qwen2.5-Coder Technical Report* (2024) | **Citations:** 3,044 (OpenAlex)
  * **Checks:** Year: PASS | Citations: PASS | Source: PASS | No Hallucination: PASS
* **Test 5 — Narrow Topic + Exact Year:** `PASS`
  * **Paper:** *Chain-Of-Thought Prompting Elicits Reasoning in Large Language Models* (2022) | **Citations:** 39,078 (OpenAlex)
  * **Checks:** Year: PASS | Citations: PASS | Source: PASS | No Hallucination: PASS
* **Test 6 — Broad + Before Year (Classic AI):** `PASS`
  * **Paper:** *Human-level control through deep reinforcement learning* (2015) | **Citations:** 39,981 (OpenAlex)
  * **Checks:** Year: PASS | Citations: PASS | Source: PASS | No Hallucination: PASS
* **Test 7 — No Good Paper (Impossible Citations + Recent):** `PASS`
  * **Result:** Correctly abstained (`Match Found: False`) — no 2026 PydanticAI paper has 50,000+ citations.
  * **Checks:** Year: PASS | Citations: PASS | Source: PASS | No Hallucination: PASS
* **Test 8 — No Good Paper (Anachronistic Topic):** `FAILED`
  * **Error:** `All models from FallbackModel failed (2 sub-exceptions)` after retrieving 10 OpenAlex candidates.
  * **Checks:** Year: FAILED | Citations: FAILED | Source: FAILED | No Hallucination: FAILED
* **Test 9 — Ambiguous Request (No Explicit Numbers):** `PASS`
  * **Paper:** *A survey on large language model based autonomous agents* (2024) | **Citations:** 5,542 (OpenAlex)
  * **Checks:** Year: PASS | Citations: PASS | Source: PASS | No Hallucination: PASS
* **Test 10 — Ambiguous Request (Vague Topic):** `PASS`
  * **Paper:** *Training language models to follow instructions with human feedback* (2022) | **Citations:** 11,061 (OpenAlex)
  * **Checks:** Year: PASS | Citations: PASS | Source: PASS | No Hallucination: PASS

*(Full structured outputs, links, explanations, and uncertainty notes are saved in `eval_results.json`.)*


## Reflection

* **What worked well?**

  Every test that completed without an external LLM API outage passed all deterministic checks (8/8 completed runs, 8/10 overall). Switching from Semantic Scholar to OpenAlex eliminated data-retrieval errors and rate-limit issues, and every single OpenAlex request returned HTTP 200. The agent handled both well-known queries (Tests 2, 5, 6, 10) and impossible queries (Test 7, where it searched three times before honestly reporting Match Found: False).

* **What failed or was unreliable?**


  1. **Single-provider LLM outages (Tests 3 & 8):** Both failures occurred because FallbackModel only contained two Google Gemini Flash-Lite models. When free-tier rate limits or 503 errors hit during sequential evaluation, both models failed together.
  2. **Semantic relevance gap (Test 4):** On a narrow prompt asking for multi-agent code generation or automated debuggin, the agent returned *Qwen2.5-Coder Technical Report* (2024). While it satisfied the year, citation, and verification checks, it is a single code-LLM report rather than a multi-agent paper, this can show why deterministic checks must be paired with manual review.
  3. **Constraint drift & date awareness (Tests 1 & 9):** In Test 1, the LLM gradually relaxed `min_citations` from `100` down to `50` and `10` across retries (only passing because *AutoGen* had 4,231 citations). In Test 9, it queried `exact=2024` when asked for a "recent" paper because the current year was not injected into the system prompt.

* **How often did the agent need tool calls?**

  Across the 8 completed tests, the agent made 15 OpenAlex tool calls, averaging ~1.9 calls per test, ranging from 1 to 4 calls.

* **Did the LLM ever hallucinate?**

  Yes. Earlier in development, Semantic Scholar rate limits caused the LLM to invent citation counts from training memory. Even with OpenAlex working better, in Test 2 the LLM attempted to output a mismatched citation count for the RAG paper, @agent.output_validator immediately flagged `[Validator] REJECTED: Citation count mismatch` and forced a retry, resulting in the verified OpenAlex count of 10,595.


* **How did you prevent incorrect answers?**

  Strict system prompt instructions forbidding memory-based metadata, Python post-filtering inside `search_openalex_papers` that records verified papers in `WorkflowState`, and a PydanticAI `@agent.output_validator` that cross-checks the final paper_link and itation_count against `WorkflowState` and triggers `ModelRetry` on any mismatch.


* **What would you improve with more time?**
  1. **Cross-provider fallback & backoff:** Add a second provider (e.g., Groq or a local Ollama fallback) and exponential backoff between retries to eliminate the  failures seen in Tests 3 and 8.
  3. **Inject the current date & add an LLM-as-a-judge relevance step:** Dynamically inject the current year into the system prompt for "recent" queries (Test 9), and add an automated relevance grader to catch topic mismatches like Test 4.
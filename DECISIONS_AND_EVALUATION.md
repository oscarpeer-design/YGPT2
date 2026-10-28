# YGPT Decisions and Evaluation Log

## My Intentions with YGPT

I came up with this product while trying to understand an open-source legacy codebase. I needed a tool that would make understanding codebases easier and review more intelligent (than using Clang-tidy). I also needed said tool to not expose confidential intellectual property. These intentions led to my system requirements.

## System Requirements

1. An intelligent documentation system that explains code files concisely
2. An intelligent code review system that evaluates and identifies potential high-risk performance and security flaws in Python and C++ code, and presents the information in a concise manner that would assist a human reviewer
3. A chat functionality that would allow said reviewer to discuss findings with the model
4. Local AI and embedding models to protect user data
5. Minimised and upfront dependencies to aid deployment
6. Compatibility with smaller hardware systems

## System Architecture and Decisions

![YGPT system architecture](images/YGPT_architecture.png)

| Decision | Rationale | Trade-offs |
|---|---|---|
| Use small, local models and dependencies (0.5B model, MiniLM embedder) | User data should be protected, and dependencies should be visible upfront | User hardware imposes a cost on the quality of the model. A 1 GB model size check also blocks larger models until it is changed |
| Embed with MiniLM through ONNX, not sentence-transformers | No PyTorch, and no requests to Hugging Face at run time, so embedding is much faster | The two model files must be placed next to the code by hand |
| Call the model through the `llama-cli` program | Simple, and no language bindings are needed | The model reloads on every call. The program is found by name, so it can't be verified |
| CPU based model calls | Allows accessibility with lower-grade consumer hardware | Affects model response times. No GPU setting is passed to `llama-cli` |
| Create hard timeouts and chunk caps during review and documentation runs | On lower performance hardware we want to prevent the risks of crashes or lagging | On large codebases important parts of the system are not viewed |
| Regex-based flagging for security and performance risks. Only flagged lines are sent to the model, and a grammar (a rule file for `llama-cli`) forces its answer into `LINE / WHY / FIX` with a flagged line number | The model is weak and has a small context window, so we improve the tooling and retrieval to maximise its effectiveness | This is very brittle and requires constant updates; it is also often context agnostic. It can't catch bug classes the regexes don't describe (e.g. use-after-free) |
| Hard caps on model response lengths and scanning response strings for patterns like repeated output words | Fixes some of the worst responses without imposing significant performance costs | This is brittle and doesn't account for other errors like factual inaccuracy or bad retrieval |
| No padding when tokenising text | Excess padding caused the embedder to treat all text stubs with high similarity, causing retrieval to fail | Each text is embedded alone, which is slower. The embedding cache key ignores tokenizer settings, so I had to rename the model to force re-embedding |
| Chunk code by symbol (function, class, etc.) with Tree-sitter | Chunk boundaries are real code boundaries | Only the first 256 tokens of a long chunk are embedded |
| Security review goes file by file, ranking within each file, and gives each file a share of the time | Matches how a reviewer reads, and every file gets some coverage | No repository-wide ranking, so the riskiest code isn't necessarily reviewed first. Each model call gets at least 30 seconds, so slow calls can use up the budget and leave later files unreviewed |
| Review whole files and repositories, not diffs | A complete picture of a codebase | Slower, and not suited to pull requests. Diff review would be a separate feature |
| Documentation of system dependencies is generated deterministically | This means the user can trust that the dependencies suggested are accurate | The resulting documentation can be harder to read. Callers are matched by function name, and only 6 key functions per file are described |
| Small stubs are removed from a list of retrieved chunks (CHAT only) | This prevents smaller pieces of code from taking up space | Possible small header functions and important constants in a code base are missed |
| CHAT searches the index saved by the most recent REVIEW | CHAT needs no paths and no extra indexing step | CHAT only knows the last code that was reviewed |

## Problems encountered with the product

1. Review was riddled with false positives and hallucinated fixes
2. Documentation was often incomplete (by design only 6 key functions per file are described, and time budgets can stop a run early)
3. Retrieval used semantic similarity scores on code, which affected the quality of chunks. Two bugs made it worse: padding made short texts look alike, and CHAT kept using an old index after a new REVIEW (both fixed)
4. A small model is prone to hallucination. It invents rationales, copies code and instructions, and can't critique itself reliably
5. Review ran out of time on a large repository (30 of 100 files in 15 minutes) because each file gets a share of the budget but each model call gets at least 30 seconds
6. Regexes can't find some real bugs

Known open issues:

- Files that aren't valid UTF-8 may be skipped
- The loop detector works line by line and can lose track of loops without braces
- Duplicate files are reviewed more than once
- The related-code threshold in REVIEW (0.40) lets almost every pair through
- The syntax error check exists but is switched off
- The `llama-cli` program isn't verified

## Evaluation

What I measured:

| Check | Result |
|---|---|
| Embedding self-retrieval (does a chunk retrieve itself in the top 5?) | About 1% before the padding fix, about 64% after. The test script wasn't saved |
| Review coverage on a repository | 30 of 100 files in a 15-minute budget (one run) |
| Review precision spot check | 0 of 3 findings on `nlohmann_json.hpp` were real defects (too small a sample to give a rate) |
| Similarity scores | A junk chunk scored 0.53 and the correct chunk 0.44, so the threshold only removes the worst misses |
| Manual QA | About two weeks, with no recorded test set |

What I haven't measured:

- Precision and recall of review findings on a labelled set
- Whether retrieval finds the right code for real questions
- Whether chat answers and documentation are correct
- Time and memory per file on target hardware

Evaluation plan:

1. **Retrieval:** 20 to 30 real questions over a few repositories, each with the expected function. Measure recall at 5 and mean reciprocal rank, with and without the threshold and stub filter
2. **Review precision:** label every finding as real, not a defect, or unclear. Break results down by signal and remove weak ones
3. **Review recall:** plant known bugs (use-after-free, overflow, race, slow loop) and check they are flagged
4. **Documentation:** check dependency lists against the real code (should be 100%) and score descriptions against a short rubric
5. **Chat:** mark each answer and its Sources list as correct or not
6. **Cost:** seconds per model call and files covered within the budget

Rerun the whole set after every change to a signal, threshold or prompt.

## Steps I would take to improve the product

1. Use a larger local model (4-8 GB) and deploy on a container. This needs the 1 GB size check and the 4096-token context raised
2. Change RAG architecture to allow for larger context window with a more powerful model
3. Combine semantic searching with exact keyword searching (BM25)
4. Create a database logging prompts and responses
5. Cache review/documentation responses so the model can operate with the same context as the user asking questions
6. Call real static analysers and performance analyser programs instead of using regexes, plus sanitizers and fuzzing for memory and race condition bugs
7. Use adversarial review to check documentation and security analysis, even with industrial static analysers (which are often context-unaware). This needs a stronger model because the small model can't critique itself
8. Allow the model to tune config values (thresholds, budgets) against the evaluation set
9. Rank review work across the whole repository before spending time
10. Add diff review as a separate feature
11. Replace chunk retrieval with a structured symbol index (definitions, callers, callees, exact text) that tools or an MCP server can query
12. Fix UTF-8 handling, duplicate files and the loop detector

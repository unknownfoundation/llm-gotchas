# llm-gotchas

![A developer at a laptop blows a bubble-gum bubble full of chat and network icons, and it bursts in their face](images/banner.png)

This repo is a collection of gotcha questions about LLM systems, the kind where the obvious answer
turns out to be wrong. No term quizzes here: you trace a request step by step, from your code to
the model and back, and find the spot where it quietly goes wrong.

Each one drops you into a moment from building a product with a language model. Make your guess,
then open the answers: every one, even the wrong ones, tells you why it seemed right and what
really happens.

## What's in it for you

- **New to LLMs?** Start from the top: what a model actually reads, what it keeps between calls,
  what a token costs. Each topic explains its words first.
- **Built something already?** You'll recognise the scenes: a chat history trimmed to fit, a cache
  of answers in front of a bot, an agent that waits for a manager's approval.
- **Designing or reviewing systems?** Head for the last topic, where things break between the
  parts. The answers are short enough to paste into a review.

The answers hold on any stack: a provider's API, your own server or a ready-made app. Defaults,
limits and prices change, so each page says when it was last checked.

## Topics

From the basics to system design, in a suggested order:

1. [Tokens and tokenizers](topics/tokens-and-tokenizers/README.md): what a model reads instead of your characters, and why the same text costs a different amount per language, format and model.
2. [Context window](topics/context-window/README.md): what the model gets on one call, what takes room in it, and what happens to a request that does not fit.
3. [Inside the model](topics/inside-the-model/README.md): what happens between the request and the first token, and what sets a model's speed and memory.
4. [Generation parameters](topics/generation-parameters/README.md): what temperature, top_p, a seed and the generation cap do to the reply.
5. [Prompting](topics/prompting/README.md): what the model reads on one call and how your code builds it: roles, examples, templates and the reasoning you ask for.
6. [Structured outputs](topics/structured-outputs/README.md): which way of asking for JSON binds your fields, and what your code does with the object it gets back.
7. [Tool calling](topics/tool-calling/README.md): the round trip from a model asking for a tool to its result: who runs the tool, what the next request must carry, and what trimming and retries do to it.
8. [Cost and latency](topics/cost-and-latency/README.md): what a request bills, how a cache prefix matches, what rate limits and a batch promise, and where the time goes.
9. [Python around the model call](topics/python-around-the-model-call/README.md): the code before and after each request: async calls, timeouts, retries, parsing and the state your service keeps.
10. [Streaming](topics/streaming/README.md): a reply sent while the model writes it: what the events carry, what a time limit or a dropped connection does, and where a check on the reply fits.
11. [Images, PDFs and audio](topics/images-pdfs-and-audio/README.md): what a picture, a document or a recording becomes on the way to the model, and what each way keeps, loses and costs.
12. [Embeddings and vector search](topics/embeddings-and-vector-search/README.md): what a similarity score tells you, and what a store returns once you add a filter, switch the model or index approximately.
13. [Retrieval and RAG](topics/retrieval-and-rag/README.md): the code between a question and the prompt: units, scores and ids, and where a wrong answer starts before any model is called.
14. [Memory](topics/memory/README.md): what an assistant seems to remember: what your code or an app writes down, when, and what comes back in a later request.
15. [Agents and workflows](topics/agents-and-workflows/README.md): the loop your code runs around a model: what a step costs, where its state lives, and what a retry, a restart or a resume repeats.
16. [Security](topics/security/README.md): why text that reaches a model can steer it, and which limits in your code decide what an injected instruction can reach.
17. [Agent protocols](topics/agent-protocols/README.md): MCP and A2A: what crosses between an application, its tool servers and other agents, and who trusts whom.
18. [LLMs on your codebase](topics/llms-on-your-codebase/README.md): what a coding assistant, a review bot or a coding agent gets from your repository, and what green tests and its commands prove or reach.
19. [Testing and evals](topics/testing-and-evals/README.md): testing the code around a model and scoring its answers: fakes, eval sets, graders and what a pass rate can show.
20. [Releasing LLM changes](topics/releasing-llm-changes/README.md): shipping a new prompt, model or setting, and what a rollback, a flag, a canary and a split test show.
21. [Where models run](topics/where-models-run/README.md): from its maker's machines to a cloud platform or your own hardware, and what each place changes about limits, versions and capacity.
22. [Data and privacy](topics/data-and-privacy/README.md): where personal data goes once it enters a request, and which copies stay behind in logs, caches, indexes and stored replies.
23. [Fine-tuning](topics/fine-tuning/README.md): training a model further on your own data: what it changes, what it needs and what it can break.
24. [AI system design](topics/ai-system-design/README.md): the parts around a model, from the gateway and the queue to the cache and the guard checks, and the failures that appear only where they meet.

## License

MIT. See [LICENSE](LICENSE).

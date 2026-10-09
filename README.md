# LangGraph Research Assistant

A multi-agent research assistant built with [LangGraph](https://github.com/langchain-ai/langgraph). Given a topic, it generates a team of AI analyst personas, lets a human review them, and then has each analyst interview an "expert" backed by live web search to produce a section of a research report.

## How it works

### 1. Create analysts (`create_analysts` graph)

```
START → create_analysts → human_feedback ──(feedback?)──→ create_analysts
                                        └──(approved)──→ END
```

- **create_analysts**: GPT-4o generates `max_analysts` personas for the topic, using structured output (Pydantic `Analyst` / `Perspectives`).
- **human_feedback**: pauses the graph with `interrupt()` so you can review the analysts. Reply with feedback to regenerate them, or `okay` / `continue` / `yes` to approve.

### 2. Interview an expert (`answer_question` graph)

```
START → ask_question → search_web  ─┐
                    └→ search_web2 ─┴→ answer_question ──→ ask_question (loop)
                                                       └─→ save_interview → write_section → END
```

- **ask_question**: the analyst asks a question that matches their persona.
- **search_web / search_web2**: an LLM turns the conversation into a search query and Tavily fetches the top results. Both run in parallel.
- **answer_question**: the expert answers using the retrieved context.
- **save_interview**: saves the transcript.
- **write_section**: writes a report section from the interview.

The loop ends after `max_num_turns` expert answers, or when the analyst says "Thank you so much for your help".

### 3. Full research agent (`research_agent` graph)

```
START → create_analysts → human_feedback ──(feedback)──→ create_analysts
                                        └──(approved, Send() per analyst)──→ conduct_interview ×N (parallel)
                                                                                     │
                                         ┌───────────────┬──────────────────────────┤
                                  write_introduction  write_report        write_conclusion
                                         └───────────────┴──────────────────────────┘
                                                              ↓
                                                     finalize_report → END
```

- **initiate_all_interviews**: once the analysts are approved, uses the `Send()` API to launch one interview sub-graph per analyst in parallel (the *map* step).
- Each interview returns a section, and `sections` (`Annotated[list, operator.add]`) collects them all (the *reduce* step).
- **write_introduction / write_report / write_conclusion** run in parallel, then **finalize_report** combines them into the final report.

## Project structure

```
├── langgraph.json            # Graph registry for `langgraph dev`
├── pyproject.toml            # Dependencies (managed with uv)
└── src/
    ├── agent.py              # Full research agent (main graph)
    ├── creating_analysts.py  # Analyst generation + human-in-the-loop graph
    ├── answering_question.py # Interview sub-graph
    └── utils/
        ├── states.py         # Graph state definitions
        ├── nodes.py          # Node functions
        ├── edges.py          # Conditional edges / routers
        ├── objects.py        # Pydantic models (Analyst, Perspectives, SearchQuery)
        ├── prompts.py        # Prompt templates
        └── models.py         # LLM configuration
```

## Setup

Requires Python 3.12 and [uv](https://docs.astral.sh/uv/).

```bash
git clone https://github.com/AbhishekKalia1103/langGraph_Agent.git
cd langGraph_Agent
uv sync
cp .env.example .env   # then add your API keys
```

You need API keys for:
- [OpenAI](https://platform.openai.com/): the LLM
- [Tavily](https://tavily.com/): web search
- [LangSmith](https://smith.langchain.com/): tracing and Studio (optional)

## Run

```bash
uv run langgraph dev
```

This starts a local API at `http://127.0.0.1:2024` and opens LangGraph Studio. Pick a graph from the dropdown.

Example input for `research_agent` or `create_analysts`:

```json
{ "topic": "The benefits of adopting LangGraph as an agent framework", "max_analysts": 3 }
```

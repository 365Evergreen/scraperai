# Copilot instructions — ScraperAI

Purpose: quickly orient a coding agent to the repository structure, runtime flows, and developer workflows so edits are safe and productive.

Big picture
- CLI-driven interactive app (entry: `scraperai/cli/app.py`) that builds a `ScraperConfig` and runs scraping flows via `Controller.run()` ([scraperai/cli/controller.py](scraperai/cli/controller.py)).
- Parser layer (`scraperai/parsers/*` + `scraperai/parsers/parserai.py`) orchestrates LLM-based detectors (page type, pagination, catalog item, data fields). `ParserAI` composes three LLM adapters (`JsonOpenAI`, `VisionOpenAI`, `PythonCodeOpenAI` in `scraperai/llm/openai.py`).
- Crawlers implement `BaseCrawler` (`scraperai/crawlers/base.py`). Two main concrete crawlers live here: `SeleniumCrawler` (`scraperai/crawlers/selenium.py`) for interactive browsing/screenshots, and `RequestsCrawler` (`scraperai/crawlers/requests.py`) for simple HTTP fetches.
- Scraper runtime (`scraperai/scraper.py`) consumes `ScraperConfig` (pydantic models in `scraperai/models.py`) and a `BaseCrawler` to produce rows via `scrape_catalog_items()` or `scrape_nested_items()`.

Key files to inspect for examples and edit points
- `scraperai/models.py` — pydantic schemas for `ScraperConfig`, `Pagination`, `WebpageFields` (source of truth for data shapes).
- `scraperai/cli/controller.py` — orchestrates full user flow (init crawler, detect page type, detect pagination/card/fields, scrape, export). Use this for change impact analysis.
- `scraperai/parsers/parserai.py` — high-level LLM orchestration, cost accounting (`total_cost`), image compression before vision calls.
- `scraperai/llm/openai.py` — concrete lm adapters; inject mocks here for offline tests or to replace vendor SDKs.
- `scraperai/crawlers/selenium.py` and `scraperai/crawlers/requests.py` — show different pagination handling and screenshot availability.
- `scraperai/scraper.py` — implements iteration & pagination logic used by export and progress bars.

Project-specific conventions & patterns
- Configuration objects are pydantic models; prefer `ScraperConfig(**payload)` and `model_dump_json()` when persisting.
- Crawler abstraction: prefer changes to `BaseCrawler` and concrete implementations for new navigation behaviors (e.g., Playwright). Many modules check crawler type (isinstance(SeleniumCrawler)) — maintain backwards-compatible APIs (`get`, `page_source`, `switch_page`).
- LLM integration is centralized in `ParserAI`; add new detectors by composing existing `*Detector` classes under `scraperai/parsers` and exposing methods on `ParserAI`.
- Use `ParserAI` constructor injection to supply test doubles for LM clients (see `ParserAI.__init__`).

Developer workflows & run commands
- Install: `pip install -r requirements.txt` or `pip install .` from repo root.
- CLI local run: `python -m scraperai.cli.app` or after install use `scraperai --url <URL>`; `OPENAI_API_KEY` can be provided via env or `.env` (controller writes `.env` when asked).
- Tests: run `pytest -q` from repo root. Some integration tests require `OPENAI_API_KEY` and `SELENOID_URL` (set in `.env` or env); unit tests under `tests/` use `tests/data/` fixtures.

Integration points to be careful with
- OpenAI/langchain: `scraperai/llm/openai.py` uses `ChatOpenAI` from `langchain_openai` and `get_openai_callback()`; edits can affect cost tracking and response formats.
- Webdrivers/Selenoid: remote webdriver management under `scraperai/crawlers/webdriver/manager.py` expects Selenoid `/status` endpoint. Tests may assume `SELENOID_URL` environment variable.
- Screenshots: `SeleniumCrawler.get_screenshot_as_base64()` is used by vision detectors — changes to encoding/compression affect prompts in `ParserAI`.

Editing guidance for agents
- Preserve pydantic schemas and use `model_dump_json()` when serializing configs; changing field names requires coordinated updates to CLI views and tests.
- When modifying detectors, add or update the corresponding methods on `ParserAI` so the CLI controller can call them without changing orchestration logic.
- Prefer DI for external services (LLM clients, webdriver manager) so tests can inject mocks; `ParserAI` and `WebdriversManager` already support this pattern.

What I couldn't discover automatically
- CI test matrix details (GitHub Actions workflows referenced in README), and the exact package entrypoint mapping for `scraperai` console script. Verify `setup.py` for console entrypoints if adding CLI changes.

If anything here is unclear or you'd like the instructions expanded with code snippets (examples of a test mock for `JsonOpenAI`, or a small refactor template for adding a new crawler), tell me which area to expand.

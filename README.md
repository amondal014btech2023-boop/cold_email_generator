# Cold Email Generator — Job Scraper + LLM Extraction

An end-to-end pipeline that scrapes job postings from company career pages, extracts structured job details (role, experience, skills, description) using an LLM, and matches them against a portfolio to generate personalized cold emails.

## Features

- **Web scraping** of career pages using `WebBaseLoader` (static HTML) and `SeleniumURLLoader` (JS-rendered pages)
- **Structured extraction** of job postings into JSON (`role`, `experience`, `skills`, `description`) via an LLM chain
- **Portfolio matching** using ChromaDB vector search to find relevant projects/skills for a given job posting
- **Cold email generation** based on matched job + portfolio context
- Built with LangChain + Groq (`llama` / `openai/gpt-oss` models via Groq API)

## Tech Stack

| Component | Tool |
|---|---|
| LLM inference | Groq API (via `langchain_groq`) |
| Orchestration | LangChain (`langchain_core`, `langchain_community`) |
| Web scraping | `WebBaseLoader`, `SeleniumURLLoader`, `trafilatura` |
| Vector store | ChromaDB |
| Environment | Python (conda env recommended) |

## Setup

### 1. Clone and create environment

```bash
conda create -n email python=3.11
conda activate email
```

### 2. Install dependencies

```bash
pip install langchain langchain-groq langchain-community chromadb selenium trafilatura webdriver-manager beautifulsoup4 python-dotenv
```

### 3. Set up your API key (never hardcode this)

Create a `.env` file in the project root:

```
GROQ_API_KEY=your_key_here
```

Load it in code:

```python
import os
from dotenv import load_dotenv

load_dotenv()
groq_api_key = os.environ["GROQ_API_KEY"]
```

> ⚠️ Never commit `.env` or paste API keys directly into notebooks/code. Add `.env` to `.gitignore`. If a key is ever exposed, rotate it immediately at [console.groq.com/keys](https://console.groq.com/keys).

## Usage

### Scrape a job posting

```python
from langchain_community.document_loaders import WebBaseLoader

loader = WebBaseLoader("https://careers.nike.com/supervisor-nike-watertown/job/R-88215")
page_data = loader.load().pop().page_content
```

For JavaScript-rendered pages where `WebBaseLoader` returns incomplete/boilerplate content, fall back to Selenium + `trafilatura` (strips nav/promo junk, keeps main content):

```python
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from webdriver_manager.chrome import ChromeDriverManager
import trafilatura, time

def scrape_job_page(url, wait_time=6):
    options = webdriver.ChromeOptions()
    options.add_argument("--headless=new")
    driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()), options=options)
    driver.get(url)
    time.sleep(wait_time)
    html = driver.page_source
    driver.quit()
    return trafilatura.extract(html, include_comments=False, include_tables=True)
```

### Extract structured job data

```python
from langchain_core.prompts import PromptTemplate
import json

prompt_extract = PromptTemplate.from_template(
    """
    ### SCRAPED TEXT FROM WEBSITE:
    {page_data}
    ### INSTRUCTION:
    The scraped text is from the career's page of a website.
    Your job is to extract the job posting and return it in JSON format containing the
    following keys: `role`, `experience`, `skills` and `description`.
    Only return the valid JSON.
    ### VALID JSON (NO PREAMBLE):
    """
)

chain_extract = prompt_extract | llm
res = chain_extract.invoke(input={'page_data': page_data})

clean_content = res.content.strip().removeprefix("```json").removesuffix("```").strip()
job_json = json.loads(clean_content)
```

### Scrape multiple postings

```python
job_urls = [
    "https://careers.nike.com/lead-process-analyst/job/R-88191",
    "https://careers.nike.com/software-engineer-ii-itc/job/R-77036",
]

all_jobs = []
for url in job_urls:
    try:
        page_data = WebBaseLoader(url).load().pop().page_content
        res = chain_extract.invoke(input={'page_data': page_data})
        clean = res.content.strip().removeprefix("```json").removesuffix("```").strip()
        all_jobs.append(json.loads(clean))
    except Exception as e:
        print(f"Failed on {url}: {e}")
```

## Project Structure

```
.
├── .env                    # API keys (not committed)
├── job_scraper.py          # scraping logic (WebBaseLoader / Selenium)
├── extract_jobs.py         # LLM-based JSON extraction chain
├── portfolio_matcher.py    # ChromaDB vector search over portfolio
├── generate_email.py       # cold email generation chain
└── README.md
```

## Known Limitations

- **Job URLs expire.** Nike (and similar ATS-backed career sites) redirect expired/filled job IDs to the homepage. Always verify the URL resolves to an actual posting before scraping.
- **JS-rendered pages** need Selenium/Playwright, not plain `WebBaseLoader`/`requests` — static scraping only sees the initial HTML shell.
- **Not all postings contain all fields.** Retail/store-level roles often omit explicit "experience"/"skills" sections; corporate/tech roles are more reliable for full extraction. Downstream logic should handle empty fields gracefully rather than assume a bug.
- **Model availability on Groq changes.** Hardcoded model names (e.g. `llama-3.1-70b-versatile`) can be deprecated without warning — check [console.groq.com/docs/models](https://console.groq.com/docs/models) periodically, or fetch `/openai/v1/models` at runtime instead of hardcoding.

## Roadmap

- [ ] Auto-extract job URLs from listing pages instead of hardcoding
- [ ] Add retry/backoff for scraping failures
- [ ] Support additional career sites beyond Nike
- [ ] Deploy as a Streamlit app for interactive use

## License

MIT

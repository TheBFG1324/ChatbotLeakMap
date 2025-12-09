# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ChatbotLeakMap is an offensive security proof-of-concept for studying multi-step prompt injection attacks against AI chatbots. It crawls and discovers chatbot APIs, classifies their domain, runs chained prompt injection attacks, and stores results in a Neo4j graph database. The system uses embedding similarity and temperature-controlled exploration to evolve attack strategies.

## Commands

### Running with Docker (recommended)
```bash
# Full deployment - starts Neo4j, initializes data, launches mock chatbots, and runs exploit
docker-compose up --build

# Rerun exploit with custom parameters (reuses running containers)
docker-compose run --rm exploit --temperature 0.8 --max-iterations 7 --threshold 0.6
```

### Running locally
```bash
# Install dependencies
pip install -r requirements.txt

# Run exploit CLI directly
python -m scripts.exploit_cli --urls http://localhost:5010 http://localhost:5011 --temperature 0.5 --max-iterations 10 --threshold 0.5

# Initialize Neo4j vector index
python -m scripts.setup_neo4j

# Load mock data
python -m scripts.mock_data
```

### Running tests
```bash
pytest test/
pytest test/crawler_test.py  # single test file
```

## Architecture

### Core Attack Flow
1. `ChatbotCrawler` (crawler/crawler.py) discovers chatbot endpoints and classifies domains via `classify_bot` (crawler/classifier.py)
2. `Exploit` (exploit/exploit.py) orchestrates the attack loop with temperature-based mode selection
3. `Exploiter` (exploit/exploiter.py) generates attack prompts using GPT-4 with a red-team system prompt
4. `Audit` (exploit/audit.py) checks responses for confidential data leaks
5. `Neo4jDriver` (db/neo4j_driver.py) stores bots, chains, prompts, and responses as a graph
6. `get_embedding` (db/embed.py) creates vector embeddings for semantic similarity search

### Key Parameters
- **temperature**: Controls explorative (novel prompts) vs exploitative (adapt past successes) mode selection
- **threshold**: Embedding similarity threshold for suggesting next prompts from successful chains
- **max_iterations**: Maximum attack steps per chain

### Neo4j Graph Schema
- **Nodes**: `Bot`, `Chain`, `Prompt`, `Response` (with vector embeddings)
- **Relationships**: `TARGETED_IN`, `STEP` (with step_number), `RESULTED_IN`, `ENDED_IN_SUCCESS`, `REFERENCED`
- Vector index `responseEmbeddingIdx` enables semantic similarity queries for prompt suggestion

### Sample Chatbots
Mock chatbots in `samples/` directory serve as attack targets:
- `educational/educational_site.py` - Port 5009
- `finance/finance_site.py` - Port 5010
- `medical/medical_site.py` - Port 5011

## Environment Variables

Required in `.env`:
- `OPENAI_API_KEY` - For embeddings and LLM calls
- `NEO4J_URI`, `NEO4J_USER`, `NEO4J_PASSWORD` - Database connection
- `BTCBANK_PROMPT`, `MEDICORP_PROMPT`, `KUEDU_PROMPT` - System prompts for mock chatbots
- `LEAK_CHECKER_PROMPT`, `ATTACK_CHECKER_PROMPT`, `EXPLOITER_PROMPT` - Prompts for audit and attack generation

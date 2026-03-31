# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Run

```bash
# Build
./mvnw clean package

# Run with a specific AI profile (e.g. claude, mistralai, openai, gemma, llama, deepseek)
# Load API keys from .env first
export $(grep -v '^#' .env | xargs) && ./mvnw spring-boot:run -Dspring-boot.run.profiles=openai

# Run tests
./mvnw -B test

# Build native image
./mvnw -Pnative -DskipTests -B spring-boot:build-image
```

Tests require Docker (uses Testcontainers for Redis and Ollama).

## Environment Variables

Set the API key for the chosen profile before running:
- `ANTHROPIC_API_KEY` — for `claude` profile
- `MISTRALAI_API_KEY` — for `mistralai` profile (default profile)
- `OPENAI_API_KEY` — for `openai` profile
- `GROQ_API_KEY` — for `gemma` / `llama` profiles
- `DEEPSEEK_API_KEY` — for `deepseek` profile
- `CHESS_ENGINE` — `stockfishonline` (default), `chessapi`, or `none`

## Architecture

This is a Spring Boot 4.0.4 / Java 21 web app where users play chess against an LLM, assisted by an optional chess engine.

**Request flow:**
1. Browser sends moves via REST (HTMX) and receives real-time board updates over WebSocket (SockJS + STOMP).
2. `BoardController` handles move submission and delegates to `AIDialogController` for async AI move generation.
3. The AI call goes through Spring AI (`AIConfig`) using a `ChatClient` configured per active profile.
4. `ChessGameTools` exposes LLM tool functions (current board state, legal moves, PGN) so the model can inspect the game.
5. `ChessEngine` (pluggable interface) validates/suggests moves independently of the LLM. Implementations live in `impl/stockfishonline/`, `impl/chessapi/`, `impl/noop/`.
6. `Board` (a record) is the central game state object, persisted in Redis via `BoardRepository`.

**AI profiles** are Spring profiles — each profile has its own `application-<profile>.properties` file that sets `spring.ai.model.chat=<provider>` to activate the right chat model. All other model types (embedding, image, etc.) are disabled globally in `application.properties`.

**System prompt** is in `src/main/resources/system-message.st` (StringTemplate format). Edit this to change how the AI approaches the game.

**Frontend** uses Thymeleaf templates + HTMX + Bootstrap. No separate frontend build step required.

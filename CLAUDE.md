# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AITradeGame is an AI-powered cryptocurrency trading simulator that uses large language models (LLMs) to make trading decisions. It supports both local desktop and online versions, with a Flask web interface and SQLite database for data persistence.

## Core Architecture

The application follows a modular design with clear separation of concerns:

### Main Components

1. **app.py** - Flask application entry point
   - REST API endpoints for providers, models, portfolios, trades, and market data
   - Background trading loop runs every 3 minutes (line 377)
   - Server starts on port 5001 by default (line 632)
   - Auto-opens browser at http://localhost:5000

2. **ai_trader.py** - LLM integration layer
   - Multi-provider support: OpenAI, DeepSeek, Anthropic (Claude), Gemini
   - Builds trading prompts with market data, portfolio, and account status
   - Parses LLM responses (expects JSON format with trading signals)
   - Trading signals: `buy_to_enter`, `sell_to_enter`, `close_position`, `hold`

3. **trading_engine.py** - Core trading logic
   - Executes trading cycles: fetch market → get portfolio → call AI → execute decisions
   - Handles position management with leverage (1-20x)
   - Calculates and applies trading fees (0.1% default, configurable)
   - Records all trades and account value snapshots

4. **database.py** - SQLite data layer
   - Tables: providers, models, portfolios, trades, conversations, account_values, settings
   - Providers table stores API configurations for different LLM services
   - Models table links trading instances to specific LLM models
   - Portfolio tracking with long/short positions and leverage support

5. **market_data.py** - Market data fetching
   - Primary: Binance API for real-time prices
   - Fallback: CoinGecko API
   - Supported coins: BTC, ETH, SOL, BNB, XRP, DOGE
   - Technical indicators: SMA (7/14 day), RSI (14 day)
   - 5-second cache to avoid rate limits

### Data Flow

```
Trading Cycle (every 3 min):
MarketDataFetcher → TradingEngine → AITrader (LLM) → TradingEngine (execute) → Database
                                                                                   ↓
                                                                         Flask API ← Frontend
```

### Key Design Patterns

- **Provider Pattern**: AI services abstracted through provider configurations, allowing multiple LLM APIs
- **Repository Pattern**: Database class encapsulates all data access
- **Singleton Pattern**: Single MarketDataFetcher instance with caching
- **Background Worker**: Daemon thread for autonomous trading loop

## Development Commands

### Setup and Run

```bash
# Install dependencies
pip install -r requirements.txt

# Run development server (starts on port 5001)
python app.py

# Run with Docker
docker-compose up -d

# Stop Docker
docker-compose down
```

### Database

The application uses SQLite with file `AITradeGame.db` created in the project root. Database schema is automatically initialized on first run via `db.init_db()` in app.py:600.

## Important Technical Details

### Trading Fee System

- Default rate: 0.1% (0.001) per trade side
- Fees charged on both entry and exit
- Stored in trades.fee column
- Configurable via `/api/settings` endpoint
- See trading_engine.py:127-145 for fee calculation

### Position Management

- Positions stored uniquely by (model_id, coin, side)
- Side can be 'long' or 'short'
- Leverage range: 1-20x
- Margin calculation: `quantity * price / leverage`
- Required capital: `margin + trading_fee`

### Portfolio Calculation Logic (database.py:164-242)

```
cash = initial_capital + realized_pnl - margin_used
unrealized_pnl = sum of position P&L based on current prices
total_value = initial_capital + realized_pnl + unrealized_pnl
```

### API Provider System

The system supports multiple LLM providers through a unified interface:
- Each provider has: name, api_url, api_key, models list
- Models reference their provider via provider_id foreign key
- OpenAI-compatible providers (OpenAI, DeepSeek) share the same client code
- Custom implementations for Anthropic and Gemini

### LLM Prompt Format

AI models receive market data, current positions, account status, and trading rules. They must respond with JSON in this format:

```json
{
  "COIN": {
    "signal": "buy_to_enter|sell_to_enter|hold|close_position",
    "quantity": 0.5,
    "leverage": 10,
    "confidence": 0.75,
    "justification": "reason"
  }
}
```

## Configuration

Settings are stored in the database (settings table) and include:
- `trading_frequency_minutes`: AI decision interval (default 60 min, though code uses 180s loop)
- `trading_fee_rate`: Commission per trade (default 0.001 = 0.1%)

Note: The trading loop in app.py:377 uses a hardcoded 180-second interval, not the database setting.

## Frontend

- Location: templates/ and static/ directories
- Built with vanilla JavaScript
- Uses ECharts for visualization
- Communicates with Flask backend via REST API

## Building Standalone Executable

The project includes PyInstaller in requirements.txt for creating Windows executables:

```bash
pyinstaller --onefile --windowed --name AITradeGame app.py
```

## Common Gotchas

1. **Port Mismatch**: Server runs on port 5001 (app.py:632) but browser opens to localhost:5000
2. **API Base URL Handling**: ai_trader.py automatically appends `/v1` to OpenAI-compatible URLs (line 101-106)
3. **Provider Type Detection**: The `provider_type` field determines which API client to use, but it's derived from provider name/URL patterns
4. **Trading Loop Frequency**: Hardcoded to 180 seconds in trading_loop(), independent of database settings
5. **Fee Application**: Fees are deducted from cash twice per round-trip trade (entry + exit)

## Database Schema Notes

- The `provider_type` field is referenced in code but not explicitly in the schema; it's likely stored in the `name` field or derived
- Models are soft-linked to providers; deleting a provider doesn't cascade delete models
- Account value history grows unbounded; consider cleanup for long-running instances
- Conversations table stores full prompt/response pairs for debugging

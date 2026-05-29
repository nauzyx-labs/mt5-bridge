# MT5 Bridge API

## Overview
`mt5-bridge` is a FastAPI service and CLI tool that mediates market data access and order execution between a MetaTrader 5 terminal and external applications.

- **Server**: Runs on Windows (where MT5 is installed) and exposes a REST API + WebSocket.
- **Client**: Runs on any platform (Windows, Linux, macOS) and communicates with the Server to fetch data or execute trades.

## Prerequisites
- Python 3.11+ (3.12 recommended — 3.14 has numpy DLL issues)
- **Server Mode**: Windows environment with MetaTrader 5 terminal installed.
- **Client Mode**: Any OS.

## Installation

### From PyPI (Recommended)

```bash
pip install mt5-bridge
```

### From Source (Development)

This project uses [uv](https://github.com/astral-sh/uv) for package management.

```bash
# Clone to Windows file system (NOT UNC/WSL paths)
cd C:\Users\you
git clone git@github.com:nauzyx-labs/mt5-bridge.git
cd mt5-bridge

# Install with Python 3.12
uv venv --python 3.12
uv sync
```

> **Note**: If you are using WSL, please checkout this repository on the **Windows file system** (e.g., `C:\Work\mt5-bridge`) and run the command from PowerShell/Command Prompt. Running Windows Python directly against a directory inside WSL (UNC path like `\\wsl.localhost\Ubuntu\...`) often causes `DLL load failed` errors with libraries like NumPy.

## Usage

### 1. Start the Server (Windows Only)

```powershell
# Default (localhost:8000)
uv run mt5-bridge server

# Custom port
uv run mt5-bridge server --port 8989
```

Additional options:
- `--mt5-path "C:\Path\To\terminal64.exe"`: Specify MT5 terminal path
- `--no-utc`: Disable Server Time -> UTC conversion

### 2. Use the Client (Any Platform)

```bash
# Health check
uv run mt5-bridge client health

# Account info
uv run mt5-bridge client account

# Get rates (M1, last 1000 bars)
uv run mt5-bridge client rates XAUUSD --timeframe M5 --count 100

# Get latest tick
uv run mt5-bridge client tick XAUUSD

# Open positions
uv run mt5-bridge client positions

# Send order
uv run mt5-bridge client order XAUUSD BUY 0.01 --sl 2000.0 --tp 2050.0

# Close position
uv run mt5-bridge client close 12345678

# Deal history (last 30 days)
uv run mt5-bridge client history --days 30 --symbol XAUUSD

# Closed positions (grouped by position_id)
uv run mt5-bridge client history_positions --days 30 --symbol XAUUSD
```

## REST API

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/health` | MT5 connection status |
| GET | `/symbols` | List available symbols |
| GET | `/rates/{symbol}?timeframe=M1&count=1000` | Historical OHLCV candles |
| GET | `/rates_range/{symbol}?timeframe=M1&start=...&end=...` | Candles by date range |
| GET | `/tick/{symbol}` | Latest tick (bid/ask) |
| GET | `/book/{symbol}` | Market depth (Level 2) |
| GET | `/ticks_from/{symbol}?start=...&count=1000&flags=ALL` | Historical ticks from date |
| GET | `/ticks_range/{symbol}?start=...&end=...&flags=ALL` | Historical ticks in range |
| GET | `/account` | Account balance, equity, margin |
| GET | `/positions?symbols=XAUUSD&magic=123456` | Open positions |
| GET | `/history?days=30&symbol=XAUUSD` | Deal history |
| GET | `/history/positions?days=30&symbol=XAUUSD` | Closed positions (grouped) |
| POST | `/order` | Send market order |
| POST | `/close` | Close position by ticket |
| POST | `/modify` | Modify SL/TP |

## WebSocket Streaming

Connect to `ws://localhost:8000/ws/stream` for real-time data push.

### Channels

| Channel | Interval | Description |
|---------|----------|-------------|
| `tick` | 0.5s | Latest bid/ask price |
| `positions` | 1s | Open positions with live PnL |
| `account` | 1s | Balance, equity, margin |
| `candles` | 1s | Latest M5 candle (OHLCV) |

### Connect

```
ws://localhost:8000/ws/stream?channels=tick,positions,account,candles
```

### Message Format

```json
{"channel": "tick", "symbol": "XAUUSD", "data": {"bid": 2350.0, "ask": 2350.2, ...}}
{"channel": "positions", "data": [{"ticket": 123, "profit": 5.0, ...}]}
{"channel": "account", "data": {"balance": 10000, "equity": 10050, ...}}
{"channel": "candles", "symbol": "XAUUSD", "timeframe": "M5", "data": {"open": 2350, "close": 2355, ...}}
```

### Dynamic Subscriptions

Send JSON messages to change subscriptions at runtime:

```json
{"action": "subscribe", "channels": ["account"]}
{"action": "unsubscribe", "channels": ["tick"]}
{"action": "ping"}
```

### Latency

- **Tick**: ~0.5s (polls every 500ms)
- **Positions/Account/Candles**: ~1s

## Architecture

- `mt5_bridge/main.py` — CLI entry point, FastAPI server, WebSocket endpoints
- `mt5_bridge/mt5_handler.py` — MetaTrader5 Python API wrapper
- `mt5_bridge/client.py` — HTTP client for the bridge API

## License
MIT License.

# Coinbase Pro Recurring Purchase Bot - AI Agent Instructions

## Project Overview
This bot automates recurring cryptocurrency purchases on Coinbase Pro using their API. The platform doesn't natively support recurring buys, so this bot implements dollar-cost averaging (DCA) through scheduled market orders.

## Architecture

### Three-Layer Design
1. **Input Layer** (`src/orders/`): Collects user preferences via CLI or YAML
   - `InputCollector`: Abstract base class defining the collection interface
   - `CommandLineInputCollector`: Interactive CLI prompts for orders
   - `YAMLInputCollector`: Reads from `orders.yaml` file
   - `DataInputVerifier`: Validates dates, times, crypto symbols, amounts

2. **API Layer** (`src/coinbase/`): Handles Coinbase Pro authentication and operations
   - `CoinbaseExchangeAuth`: Custom HMAC SHA256 authentication (extends `requests.AuthBase`)
   - `CoinbaseProHandler`: API wrapper for deposits, orders, transaction details, email confirmations
   - `CoinbaseBot`: Orchestrates the scheduling and execution loop

3. **Entry Point** (`place_order.py`): Main driver that wires everything together

### Key Data Flow
```
User Input (CLI/YAML) → InputCollector.collect_inputs() → 
CoinbaseBot.__init__(frequency, start_date, start_time, orders) →
CoinbaseBot.activate() infinite loop →
  if time_to_deposit(): deposit_from_bank() →
  if time_to_purchase(): place_market_order() for each crypto
```

## Critical Setup Workflow

### Initial Setup (one-time)
```bash
# 1. Edit initialize.sh with API credentials
# 2. Run initialization script
sh initialize.sh
```

The script:
- Creates Python virtual environment in `env/`
- Installs dependencies from `requirements.txt`
- Sets up pre-commit hooks (black, isort)
- Generates `.env` file from variables in initialize.sh

### Environment Variables
All secrets stored in `.env` (gitignored):
- `CB_API_KEY`, `CB_API_SECRET`, `CB_API_PASS`: Production Coinbase Pro credentials
- `CB_API_KEY_TEST`, `CB_API_SECRET_TEST`, `CB_API_PASS_TEST`: Sandbox credentials for testing
- `EMAIL_ADDRESS`, `EMAIL_PASSWORD`: Gmail SMTP for order confirmations

Loaded via `python-dotenv` in `src/coinbase/utilities.py` as credential classes.

## Running the Bot

### CLI Mode (default)
```bash
python place_order.py
# Interactive prompts for: start_date, start_time, frequency, crypto symbols, amounts
```

### YAML Mode
```bash
python place_order.py --yaml
# Reads from orders.yaml in project root
```

**YAML Structure:**
```yaml
crypto: [BTC, ETH]        # List of symbols
start_date: 2025-01-01    # YYYY-MM-DD
start_time: 07:00 AM      # HH:MM AM/PM
frequency: weekly         # daily|weekly|biweekly|monthly
amount_usd: [100, 50]     # Must match crypto list length
```

## Project-Specific Patterns

### Frequency Handling
`FREQUENCY_TO_DAYS` dict in `src/coinbase/frequency.py` maps strings to `timedelta`/`relativedelta`:
- "daily" → 1 day, "weekly" → 7 days, "biweekly" → 14 days, "monthly" → 1 month (handles variable days)

### Sandbox vs Production
Two API URLs defined:
- Production: `https://api.pro.coinbase.com/`
- Sandbox: `https://api-public.sandbox.pro.coinbase.com/`

**Important limitations checked via `if "sandbox" not in self.coinbase.api_url:`:**
- `deposit_from_bank()`: Not supported in sandbox (prints warning)
- `are_sufficient_funds_available()`: Not supported in sandbox (prints warning)

### Transaction Timing
Bot uses two timestamps:
- `next_deposit_date`: Deposits USD 1 minute before purchase
- `next_purchase_date`: Places market orders at specified time

Infinite loop with `sleep()` checks conditions every iteration.

### Input Validation
`DataInputVerifier` enforces:
- Date/time must be ≥60 seconds in future (prevents race conditions)
- Crypto symbols validated against Coinbase API
- Dollar amounts must be positive numbers
- CLI retries on invalid input; YAML raises `RuntimeError`

## Testing Strategy

### Running Tests
```bash
pytest  # Uses config in pyproject.toml
```

All tests require **sandbox credentials** to be filled in `.env`. Tests are skipped if credentials empty:
```python
@pytest.mark.skipif(SANDBOX_CREDENTIALS.empty_credentials, reason="No API credentials provided")
```

Some tests explicitly skip production-only features:
```python
@pytest.mark.skip(reason="deposit_from_bank() is not supported in sandbox mode")
```

### Test Files Structure
- `test_coinbase_bot.py`: Bot scheduling and activation logic
- `test_coinbase_pro_handler.py`: API operations (deposits, orders, payments)
- `test_yaml_input_collector.py`: YAML parsing with valid/invalid files in `tests/files/`
- `test_command_line_input_collection.py`: CLI input validation
- `test_orders_utility_methods.py`: Date/time/crypto validation helpers

## Code Style

### Formatting (enforced by pre-commit)
- **Black**: Line length 120 (see `pyproject.toml`)
- **Isort**: Line length 120, black profile

### Naming Conventions
- Classes: PascalCase (`CoinbaseBot`, `YAMLInputCollector`)
- Methods: snake_case (`place_market_order`, `get_payment_method`)
- Constants: UPPER_SNAKE_CASE (`FREQUENCY_TO_DAYS`, `COINBASE_API_URL`)

### Error Handling
- Raise `RuntimeError` for API failures with descriptive messages
- Raise `TypeError`/`ValueError` for invalid parameters
- Print "SUCCESS:", "WARNING:", "ERROR:" prefixes for user feedback
- Email failures are non-fatal (return `False`, print warning)

## Common Tasks

### Adding a New Frequency
1. Update `FREQUENCY_TO_DAYS` in `src/coinbase/frequency.py`
2. Update validation in `DataInputVerifier.is_valid_frequency()`
3. Add test cases in `test_orders_utility_methods.py`

### Adding a New Input Method
1. Create subclass of `InputCollector` in `src/orders/`
2. Implement abstract methods: `get_start_date()`, `get_start_time()`, `get_frequency()`, `get_orders()`, `collect_inputs()`
3. Update `place_order.py` to add CLI argument and instantiate new collector

### Modifying API Interactions
- All API calls go through `CoinbaseProHandler` methods
- Authentication automatically added via `CoinbaseExchangeAuth.__call__()`
- Always check `response.status_code != 200` and raise `RuntimeError` with `response.content`

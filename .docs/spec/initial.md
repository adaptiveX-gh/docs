```bash
# .env.example

# Application
APP_NAME=freqtrade-strategy-builder
APP_VERSION=0.1.0
ENVIRONMENT=development  # development, staging, production
DEBUG=true
LOG_LEVEL=INFO

# API
API_HOST=0.0.0.0
API_PORT=8000
API_WORKERS=4
API_RELOAD=true  # development only

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/freqtrade_builder
DATABASE_POOL_SIZE=20
DATABASE_MAX_OVERFLOW=10

# Redis
REDIS_URL=redis://localhost:6379/0
REDIS_PASSWORD=

# Queue
RQ_QUEUE_NAME=backtests
RQ_WORKER_COUNT=2

# FreqTrade
FREQTRADE_DIR=/opt/freqtrade
FREQTRADE_USER_DATA=/opt/freqtrade/user_data
FREQTRADE_TIMEOUT=300

# LLM APIs
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...  # fallback
LLM_MODEL=claude-sonnet-4.5
LLM_MAX_TOKENS=4096
LLM_TEMPERATURE=0.7

# Cloud Providers
VULTR_API_KEY=
DIGITALOCEAN_TOKEN=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=

# Security
JWT_SECRET_KEY=your-secret-key-here-change-in-production
JWT_ALGORITHM=HS256
JWT_EXPIRATION_HOURS=24
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:8000

# Rate Limiting
RATE_LIMIT_ENABLED=true
RATE_LIMIT_PER_HOUR=100
RATE_LIMIT_BACKTEST_PER_DAY=50

# Storage
UPLOAD_MAX_SIZE_MB=10
UPLOAD_EXPIRATION_HOURS=24
TEMP_DIR=/tmp/freqtrade_builder

# Monitoring (optional)
SENTRY_DSN=
PROMETHEUS_ENABLED=false
PROMETHEUS_PORT=9090

# Feature Flags
ENABLE_LIVE_TRADING=false
ENABLE_COMPONENT_MARKETPLACE=true
ENABLE_AI_OPTIMIZATION=true
```

### 13.2 Database Migrations

```python
# migrations/002_add_backtest_cache.sql
-- Add caching for backtest results to avoid redundant runs

CREATE TABLE backtest_cache (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    strategy_code_hash VARCHAR(64) NOT NULL,
    config_hash VARCHAR(64) NOT NULL,
    results JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    accessed_at TIMESTAMP DEFAULT NOW(),
    access_count INTEGER DEFAULT 1,
    UNIQUE(strategy_code_hash, config_hash)
);

CREATE INDEX idx_backtest_cache_hash ON backtest_cache(strategy_code_hash, config_hash);
CREATE INDEX idx_backtest_cache_accessed ON backtest_cache(accessed_at DESC);

-- Cleanup old cache entries (run periodically)
CREATE OR REPLACE FUNCTION cleanup_old_backtest_cache()
RETURNS void AS $$
BEGIN
    DELETE FROM backtest_cache
    WHERE accessed_at < NOW() - INTERVAL '30 days';
END;
$$ LANGUAGE plpgsql;
```

```python
# migrations/003_add_strategy_versions.sql
-- Track strategy version history

CREATE TABLE strategy_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    strategy_id UUID NOT NULL REFERENCES strategies(id) ON DELETE CASCADE,
    version INTEGER NOT NULL,
    code TEXT NOT NULL,
    config JSONB NOT NULL,
    changes TEXT,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(strategy_id, version)
);

CREATE INDEX idx_strategy_versions_strategy ON strategy_versions(strategy_id);
CREATE INDEX idx_strategy_versions_created ON strategy_versions(created_at DESC);

-- Trigger to auto-increment version
CREATE OR REPLACE FUNCTION increment_strategy_version()
RETURNS TRIGGER AS $$
BEGIN
    NEW.version := (
        SELECT COALESCE(MAX(version), 0) + 1
        FROM strategy_versions
        WHERE strategy_id = NEW.strategy_id
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER strategy_version_increment
BEFORE INSERT ON strategy_versions
FOR EACH ROW
EXECUTE FUNCTION increment_strategy_version();
```

### 13.3 API Client Examples

#### Python Client

```python
# examples/python_client.py
"""
Example Python client for the Strategy Builder API
"""
import httpx
import asyncio
from typing import Dict, Any

class StrategyBuilderClient:
    """Python client for Strategy Builder API"""
    
    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url
        self.client = httpx.AsyncClient(
            headers={"Authorization": f"Bearer {api_key}"}
        )
    
    async def create_strategy(
        self,
        name: str,
        description: str,
        pairs: list[str],
        timeframe: str = "15m"
    ) -> Dict[str, Any]:
        """Create a new strategy"""
        response = await self.client.post(
            f"{self.base_url}/api/v1/strategies/create",
            json={
                "name": name,
                "description": description,
                "intent": {
                    "pairs": pairs,
                    "timeframe": timeframe
                }
            }
        )
        response.raise_for_status()
        return response.json()
    
    async def run_backtest(
        self,
        strategy_id: str,
        days: int = 90
    ) -> Dict[str, Any]:
        """Run backtest on strategy"""
        response = await self.client.post(
            f"{self.base_url}/api/v1/strategies/{strategy_id}/backtest",
            json={
                "config": {
                    "days": days,
                    "stake_amount": 100
                }
            }
        )
        response.raise_for_status()
        return response.json()
    
    async def get_backtest_results(
        self,
        strategy_id: str,
        backtest_id: str
    ) -> Dict[str, Any]:
        """Get backtest results"""
        response = await self.client.get(
            f"{self.base_url}/api/v1/strategies/{strategy_id}/backtest/{backtest_id}"
        )
        response.raise_for_status()
        return response.json()
    
    async def deploy(
        self,
        strategy_id: str,
        provider: str = "vultr",
        mode: str = "dry_run"
    ) -> Dict[str, Any]:
        """Deploy strategy to cloud"""
        response = await self.client.post(
            f"{self.base_url}/api/v1/strategies/{strategy_id}/deploy",
            json={
                "provider": provider,
                "mode": mode,
                "region": "ewr",
                "instance_type": "vc2-1c-2gb"
            }
        )
        response.raise_for_status()
        return response.json()
    
    async def close(self):
        """Close HTTP client"""
        await self.client.aclose()

# Usage example
async def main():
    client = StrategyBuilderClient(
        base_url="http://localhost:8000",
        api_key="your-api-key"
    )
    
    try:
        # Create strategy
        strategy = await client.create_strategy(
            name="My Mean Reversion",
            description="Buy when RSI is oversold",
            pairs=["BTC/USDT", "ETH/USDT"]
        )
        print(f"Created strategy: {strategy['strategyId']}")
        
        # Run backtest
        backtest = await client.run_backtest(
            strategy_id=strategy['strategyId'],
            days=90
        )
        print(f"Backtest started: {backtest['backtestId']}")
        
        # Wait and get results
        await asyncio.sleep(60)  # Wait for completion
        results = await client.get_backtest_results(
            strategy_id=strategy['strategyId'],
            backtest_id=backtest['backtestId']
        )
        print(f"Win rate: {results['results']['win_rate']:.1%}")
        
    finally:
        await client.close()

if __name__ == "__main__":
    asyncio.run(main())
```

#### cURL Examples

```bash
# examples/curl_examples.sh

# 1. Create strategy
curl -X POST http://localhost:8000/api/v1/strategies/create \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test Strategy",
    "description": "Mean reversion strategy",
    "intent": {
      "pairs": ["BTC/USDT"],
      "timeframe": "15m",
      "risk_tolerance": "moderate"
    }
  }'

# 2. Search components
curl -X GET "http://localhost:8000/api/v1/components/search?q=rsi&category=indicators" \
  -H "Authorization: Bearer $API_KEY"

# 3. Get component details
curl -X GET http://localhost:8000/api/v1/components/adaptive_rsi \
  -H "Authorization: Bearer $API_KEY"

# 4. Run backtest
curl -X POST http://localhost:8000/api/v1/strategies/$STRATEGY_ID/backtest \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "config": {
      "pairs": ["BTC/USDT"],
      "timeframe": "15m",
      "days": 90,
      "stake_amount": 100
    }
  }'

# 5. Stream backtest progress (SSE)
curl -N http://localhost:8000/api/v1/strategies/$STRATEGY_ID/backtest/$BACKTEST_ID/stream \
  -H "Authorization: Bearer $API_KEY" \
  -H "Accept: text/event-stream"

# 6. Deploy to cloud
curl -X POST http://localhost:8000/api/v1/strategies/$STRATEGY_ID/deploy \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "mode": "dry_run",
    "provider": "vultr",
    "region": "ewr",
    "instance_type": "vc2-1c-2gb"
  }'
```

### 13.4 Docker Configuration

```dockerfile
# docker/Dockerfile
FROM python:3.11-slim

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    build-essential \
    libssl-dev \
    libffi-dev \
    && rm -rf /var/lib/apt/lists/*

# Create app directory
WORKDIR /app

# Copy requirements
COPY requirements.txt .
COPY requirements-dev.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY src/ ./src/
COPY components/ ./components/
COPY pyproject.toml .

# Install application
RUN pip install -e .

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app
USER appuser

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD python -c "import httpx; httpx.get('http://localhost:8000/health')"

# Run application
CMD ["uvicorn", "src.interfaces.api.app:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker/docker-compose.yml
version: '3.8'

services:
  # PostgreSQL database
  postgres:
    image: postgres:15-alpine
    container_name: strategy_builder_db
    environment:
      POSTGRES_DB: freqtrade_builder
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
  
  # Redis for caching and queue
  redis:
    image: redis:7-alpine
    container_name: strategy_builder_redis
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
  
  # API server
  api:
    build:
      context: ..
      dockerfile: docker/Dockerfile
    container_name: strategy_builder_api
    environment:
      DATABASE_URL: postgresql://postgres:postgres@postgres:5432/freqtrade_builder
      REDIS_URL: redis://redis:6379/0
      FREQTRADE_DIR: /opt/freqtrade
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
    volumes:
      - ../src:/app/src
      - ../components:/app/components
      - freqtrade_data:/opt/freqtrade/user_data
    ports:
      - "8000:8000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: uvicorn src.interfaces.api.app:app --host 0.0.0.0 --port 8000 --reload
  
  # RQ worker for background jobs
  worker:
    build:
      context: ..
      dockerfile: docker/Dockerfile
    container_name: strategy_builder_worker
    environment:
      DATABASE_URL: postgresql://postgres:postgres@postgres:5432/freqtrade_builder
      REDIS_URL: redis://redis:6379/0
      FREQTRADE_DIR: /opt/freqtrade
    volumes:
      - ../src:/app/src
      - freqtrade_data:/opt/freqtrade/user_data
    depends_on:
      - postgres
      - redis
    command: rq worker --url redis://redis:6379/0 backtests
  
  # FreqTrade (optional, for testing)
  freqtrade:
    image: freqtradeorg/freqtrade:stable
    container_name: freqtrade
    volumes:
      - freqtrade_data:/freqtrade/user_data
    ports:
      - "8080:8080"
    command: webserver --config /freqtrade/user_data/config.json

volumes:
  postgres_data:
  freqtrade_data:
```

### 13.5 Deployment Scripts

```bash
# scripts/deploy.sh
#!/bin/bash
# Production deployment script

set -e

echo "🚀 Deploying Strategy Builder..."

# Load environment
if [ -f .env.production ]; then
    source .env.production
else
    echo "❌ .env.production not found"
    exit 1
fi

# Build Docker images
echo "📦 Building Docker images..."
docker-compose -f docker/docker-compose.yml build

# Run database migrations
echo "🗄️  Running database migrations..."
docker-compose -f docker/docker-compose.yml run --rm api alembic upgrade head

# Start services
echo "▶️  Starting services..."
docker-compose -f docker/docker-compose.yml up -d

# Wait for health checks
echo "🏥 Waiting for health checks..."
sleep 10

# Verify deployment
echo "✅ Verifying deployment..."
curl -f http://localhost:8000/health || {
    echo "❌ Health check failed"
    docker-compose -f docker/docker-compose.yml logs api
    exit 1
}

echo "✨ Deployment complete!"
echo "📊 API: http://localhost:8000"
echo "📚 Docs: http://localhost:8000/docs"
```

```bash
# scripts/backup.sh
#!/bin/bash
# Backup database and components

set -e

BACKUP_DIR="/backups/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

echo "💾 Starting backup..."

# Backup PostgreSQL
echo "Backing up database..."
docker-compose exec -T postgres pg_dump -U postgres freqtrade_builder > "$BACKUP_DIR/database.sql"

# Backup components
echo "Backing up components..."
tar -czf "$BACKUP_DIR/components.tar.gz" components/

# Backup user data
echo "Backing up user data..."
docker-compose exec -T api tar -czf - /opt/freqtrade/user_data > "$BACKUP_DIR/user_data.tar.gz"

# Create backup manifest
cat > "$BACKUP_DIR/manifest.json" <<EOF
{
  "backup_date": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "database_size": "$(du -h "$BACKUP_DIR/database.sql" | cut -f1)",
  "components_size": "$(du -h "$BACKUP_DIR/components.tar.gz" | cut -f1)",
  "user_data_size": "$(du -h "$BACKUP_DIR/user_data.tar.gz" | cut -f1)"
}
EOF

echo "✅ Backup complete: $BACKUP_DIR"
```

### 13.6 Monitoring Setup

```yaml
# docker/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'strategy_builder'
    static_configs:
      - targets: ['api:9090']
    metrics_path: '/metrics'
```

```python
# src/infrastructure/monitoring/metrics.py
"""
Prometheus metrics for monitoring
"""
from prometheus_client import Counter, Histogram, Gauge
import time
from functools import wraps

# Metrics
backtest_requests = Counter(
    'backtest_requests_total',
    'Total number of backtest requests',
    ['status']
)

backtest_duration = Histogram(
    'backtest_duration_seconds',
    'Duration of backtests',
    buckets=[10, 30, 60, 120, 300, 600]
)

active_backtests = Gauge(
    'active_backtests',
    'Number of currently running backtests'
)

strategy_creations = Counter(
    'strategy_creations_total',
    'Total number of strategies created'
)

component_downloads = Counter(
    'component_downloads_total',
    'Total component downloads',
    ['component_id']
)

# Decorators
def track_backtest_time(func):
    """Decorator to track backtest duration"""
    @wraps(func)
    async def wrapper(*args, **kwargs):
        start = time.time()
        active_backtests.inc()
        
        try:
            result = await func(*args, **kwargs)
            backtest_requests.labels(status='success').inc()
            return result
        except Exception as e:
            backtest_requests.labels(status='error').inc()
            raise
        finally:
            duration = time.time() - start
            backtest_duration.observe(duration)
            active_backtests.dec()
    
    return wrapper
```

### 13.7 CLI Tool

```python
# src/interfaces/cli/commands.py
"""
Command-line interface for Strategy Builder
"""
import click
import asyncio
from rich.console import Console
from rich.table import Table
from rich.progress import Progress

from src.core.services.strategy_builder import StrategyBuilderService
from src.core.services.component_library import ComponentLibraryService

console = Console()

@click.group()
def cli():
    """FreqTrade Strategy Builder CLI"""
    pass

@cli.command()
@click.option('--query', '-q', required=True, help='Search query')
@click.option('--category', '-c', help='Filter by category')
def search(query: str, category: str):
    """Search for components"""
    service = ComponentLibraryService()
    results = service.search(query=query, category=category)
    
    table = Table(title=f"Search Results: {query}")
    table.add_column("ID", style="cyan")
    table.add_column("Name", style="green")
    table.add_column("Category")
    table.add_column("Rating", justify="right")
    table.add_column("Downloads", justify="right")
    
    for component in results:
        table.add_row(
            component.id,
            component.name,
            component.category,
            f"{component.rating:.1f}⭐",
            f"{component.downloads:,}"
        )
    
    console.print(table)

@cli.command()
@click.option('--name', '-n', required=True, help='Strategy name')
@click.option('--description', '-d', required=True, help='Strategy description')
@click.option('--pairs', '-p', multiple=True, required=True, help='Trading pairs')
def create(name: str, description: str, pairs: tuple):
    """Create a new strategy"""
    service = StrategyBuilderService()
    
    with Progress() as progress:
        task = progress.add_task("[cyan]Creating strategy...", total=100)
        
        strategy = service.create(
            name=name,
            description=description,
            intent={
                "pairs": list(pairs),
                "timeframe": "15m"
            }
        )
        progress.update(task, advance=50)
        
        # Generate code
        asyncio.run(service.generate_code(strategy.id))
        progress.update(task, advance=50)
    
    console.print(f"✅ Strategy created: [green]{strategy.id}[/green]")
    console.print(f"📁 File: {strategy.file_path}")

@cli.command()
@click.argument('strategy_id')
@click.option('--days', '-d', default=90, help='Number of days to backtest')
def backtest(strategy_id: str, days: int):
    """Run backtest on a strategy"""
    from src.core.services.backtest_orchestrator import BacktestOrchestrator
    
    service = BacktestOrchestrator()
    
    with Progress() as progress:
        task = progress.add_task("[cyan]Running backtest...", total=100)
        
        # Start backtest
        job = service.start_backtest(
            strategy_id=strategy_id,
            config={"days": days}
        )
        
        # Poll for progress
        while True:
            status = service.get_job(job.id)
            progress.update(task, completed=status.progress)
            
            if status.status in ['completed', 'failed']:
                break
            
            asyncio.sleep(2)
    
    if status.status == 'completed':
        results = status.results
        
        table = Table(title="Backtest Results")
        table.add_column("Metric", style="cyan")
        table.add_column("Value", style="green")
        
        table.add_row("Total Trades", str(results['total_trades']))
        table.add_row("Win Rate", f"{results['win_rate']:.1%}")
        table.add_row("Sharpe Ratio", f"{results['sharpe_ratio']:.2f}")
        table.add_row("Max Drawdown", f"{results['max_drawdown']:.1%}")
        
        console.print(table)
    else:
        console.print(f"❌ Backtest failed: {status.error}", style="red")

@cli.command()
def list():
    """List all strategies"""
    service = StrategyBuilderService()
    strategies = service.list()
    
    table = Table(title="Your Strategies")
    table.add_column("ID", style="cyan")
    table.add_column("Name", style="green")
    table.add_column("Status")
    table.add_column("Created", justify="right")
    
    for strategy in strategies:
        table.add_row(
            str(strategy.id)[:8],
            strategy.name,
            strategy.status.value,
            strategy.created_at.strftime("%Y-%m-%d")
        )
    
    console.print(table)

if __name__ == '__main__':
    cli()
```

### 13.8 Performance Benchmarks

```python
# tests/performance/benchmark.py
"""
Performance benchmarks
"""
import time
import statistics
from typing import List

def benchmark_backtest_performance():
    """Benchmark backtest execution time"""
    from src.adapters.freqtrade.cli_adapter import FreqTradeCLIAdapter
    from src.core.domain.ports import BacktestConfig
    
    adapter = FreqTradeCLIAdapter(
        freqtrade_dir=Path("/opt/freqtrade"),
        user_data_dir=Path("/opt/freqtrade/user_data")
    )
    
    # Sample strategy
    strategy_code = """
from freqtrade.strategy import IStrategy

class TestStrategy(IStrategy):
    def populate_indicators(self, dataframe, metadata):
        return dataframe
    
    def populate_entry_trend(self, dataframe, metadata):
        dataframe['enter_long'] = dataframe['close'] > dataframe['open']
        return dataframe
    
    def populate_exit_trend(self, dataframe, metadata):
        dataframe['exit_long'] = dataframe['close'] < dataframe['open']
        return dataframe
    """
    
    config = BacktestConfig(
        strategy_code=strategy_code,
        pairs=["BTC/USDT"],
        timeframe="5m",
        start_date="20240101",
        end_date="20240131",
        stake_amount=100
    )
    
    # Run multiple backtests
    durations: List[float] = []
    
    for i in range(5):
        start = time.time()
        result = adapter.run_backtest(config)
        duration = time.time() - start
        durations.append(duration)
        print(f"Run {i+1}: {duration:.2f}s")
    
    print(f"\nResults:")
    print(f"  Average: {statistics.mean(durations):.2f}s")
    print(f"  Median: {statistics.median(durations):.2f}s")
    print(f"  Min: {min(durations):.2f}s")
    print(f"  Max: {max(durations):.2f}s")
    print(f"  Std Dev: {statistics.stdev(durations):.2f}s")

if __name__ == '__main__':
    benchmark_backtest_performance()
```

**Expected Performance:**
- Simple backtest (1 month, 1 pair, 5m): 15-30 seconds
- Complex backtest (1 year, 5 pairs, 15m): 2-5 minutes
- Strategy validation: < 5 seconds
- Component search: < 100ms
- API response time (p95): < 500ms

### 13.9 Troubleshooting Guide

#### Common Issues

**1. FreqTrade Not Found**
```bash
# Error: FreqTrade not found in PATH
# Solution: Install FreqTrade or add to PATH
pip install freqtrade
# Or specify path in config
FREQTRADE_DIR=/path/to/freqtrade
```

**2. Database Connection Failed**
```bash
# Error: could not connect to server
# Solution: Check PostgreSQL is running
docker-compose ps postgres
docker-compose logs postgres

# Reset database
docker-compose down -v
docker-compose up -d postgres
```

**3. Backtest Timeout**
```bash
# Error: Backtest timed out after 300s
# Solution: Increase timeout or reduce data range
FREQTRADE_TIMEOUT=600  # 10 minutes

# Or reduce backtest period
config = {"days": 30}  # instead of 90
```

**4. Out of Memory**
```bash
# Error: MemoryError during backtest
# Solution: Increase Docker memory limit
# In docker-compose.yml:
services:
  api:
    mem_limit: 4g
```

**5. Rate Limit Exceeded**
```bash
# Error: 429 Too Many Requests
# Solution: Wait or increase limits
RATE_LIMIT_PER_HOUR=200
```

### 13.10 Development Workflow

```bash
# Setup development environment
git clone https://github.com/your-org/freqtrade-strategy-builder.git
cd freqtrade-strategy-builder

# Create virtual environment
python3.11 -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt
pip install -r requirements-dev.txt
pip install -e .

# Setup pre-commit hooks
pre-commit install

# Copy environment file
cp .env.example .env
# Edit .env with your settings

# Start database
docker-compose up -d postgres redis

# Run migrations
alembic upgrade head

# Start API server
uvicorn src.interfaces.api.app:app --reload

# In another terminal, start worker
rq worker --url redis://localhost:6379/0 backtests

# Run tests
pytest

# Run linting
black src/
ruff check src/
mypy src/

# Build documentation
cd docs
mkdocs serve
```

---

## 14. Glossary

| Term | Definition |
|------|------------|
| **Adapter** | Design pattern that translates between incompatible interfaces |
| **Backtest** | Testing a trading strategy on historical data |
| **Blueprint** | High-level plan for a strategy before code generation |
| **Component** | Reusable piece of trading logic (indicator, signal, etc.) |
| **Cloud-init** | Standard for instance initialization in cloud environments |
| **Dry-run** | Paper trading mode with simulated money |
| **FreqTrade** | Open-source cryptocurrency trading bot |
| **Hexagonal Architecture** | Design pattern that separates business logic from external concerns |
| **Hyperopt** | Hyperparameter optimization in FreqTrade |
| **Indicator** | Technical analysis calculation (RSI, EMA, etc.) |
| **MCP** | Model Context Protocol - standard for AI tool integration |
| **Port** | Interface that defines a contract for external systems |
| **Sharpe Ratio** | Risk-adjusted return metric |
| **SSE** | Server-Sent Events - protocol for server push |
| **Strategy** | Set of rules for entering and exiting trades |
| **Timeframe** | Candle duration (5m, 15m, 1h, etc.) |

---

## 15. References

### Official Documentation
- [FreqTrade Documentation](https://www.freqtrade.io/en/stable/)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Anthropic Claude API](https://docs.anthropic.com/)

### Cloud Providers
- [Vultr API Documentation](https://www.vultr.com/api/)
- [DigitalOcean API](https://docs.digitalocean.com/reference/api/)
- [Cloud-init Documentation](https://cloudinit.readthedocs.io/)

### Trading Resources
- [Investopedia - Technical Analysis](https://www.investopedia.com/technical-analysis-4689657)
- [QuantConnect - Strategy Examples](https://www.quantconnect.com/docs/v2/)

---

## 16. License & Disclaimer

### License
This specification is provided under the MIT License.

### Disclaimer
```
IMPORTANT DISCLAIMER:

This software is for EDUCATIONAL PURPOSES ONLY.

Cryptocurrency trading involves substantial risk of loss. 
Do NOT risk money you cannot afford to lose.

USE THE SOFTWARE AT YOUR OWN RISK.

The authors and contributors:
- Make NO guarantees about profitability
- Accept NO responsibility for trading losses
- Provide NO investment advice
- Are NOT financial advisors

Always:
- Start with paper trading (dry-run mode)
- Test strategies thoroughly before deploying
- Never invest more than you can afford to lose
- Consult a qualified financial advisor

Past performance does NOT guarantee future results.
```

---

## Document Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-01-10 | Initial | Complete technical specification |

---

**End of Technical Specification**

This specification provides a complete blueprint for building the FreqTrade Strategy Builder. The architecture is designed to be:

1. **Modular** - Easy to extend and maintain
2. **Scalable** - Handles growth from single user to thousands
3. **Safe** - Multiple validation and safety layers
4. **Maintainable** - Loose coupling with FreqTrade allows independent updates
5. **Production-Ready** - Includes deployment, monitoring, and operations

The MVP focuses on core functionality (MCP + API + CLI) while laying groundwork for future enhancements (Web UI, marketplace, advanced features).
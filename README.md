# Custom Monitoring Solution

## Project Overview

A comprehensive, agent-based monitoring solution designed as a SaaS platform with a focus on reliability, dual-alerting redundancy, and support for specific service monitoring including Tailscale, Minio, PostgreSQL, custom applications, and Hikvision gateways.

## Core Architecture

### Three-Tier Architecture

```
┌─────────────────────────────────────────┐
│       LOCAL SERVER (Edge)               │
│  - Monitoring Agent (Daemon)            │
│  - Local SQLite Database (Buffer)       │
│  - Optional Local Dashboard             │
│  - Direct Alert Capability              │
└──────────────┬──────────────────────────┘
               ↓
┌──────────────┴──────────────────────────┐
│       CLOUD BACKEND (Supabase)          │
│  - Heartbeat Monitor                    │
│  - Health Check Pinger                  │
│  - TimescaleDB Storage                  │
│  - Alert Engine                         │
│  - Web Dashboard                        │
│  - Account Management                   │
└─────────────────────────────────────────┘
```

## Component Details

### 1. Monitoring Agent (Local Server)

**Purpose**: Edge-deployed monitoring daemon that collects metrics, monitors services, and provides first-line alerting.

**Key Features**:
- Lightweight daemon process (runs as systemd service on Linux)
- Configurable plugin-based service monitoring
- Local SQLite database for 24-hour metric buffering
- Direct alerting capability (email/Slack) for critical events
- Sends heartbeat every 60 seconds to cloud backend
- Pushes metrics every 30 seconds to cloud storage
- Optional local dashboard accessible via HTTP (e.g., port 3000)
- Works offline when cloud is unreachable

**Monitored Services** (Configurable via YAML):
- **System Metrics**: CPU, RAM, disk usage, network I/O, file system
- **Systemd Services**: Monitor service status, restarts, failures
- **Tailscale**: Connection status, peer connectivity, network health
- **Minio**: Storage utilization, bucket health, API availability
- **PostgreSQL**: Connection pool, query performance, replication lag
- **Custom C# Application**: Process health, memory usage, custom metrics
- **Hikvision Gateway**: Gateway connectivity, camera status, recording state
- **HTTP/API Endpoints**: Health checks, response times, status codes

**Configuration Format** (YAML):
```yaml
api_key: "user_api_key_xxx"
server_name: "production-server-1"
heartbeat_interval: 60s
metrics_push_interval: 30s

monitors:
  - type: system
    enabled: true
    metrics: [cpu, ram, disk, network]
    
  - type: tailscale
    enabled: true
    check_interval: 30s
    
  - type: postgres
    enabled: true
    connection: "postgres://localhost:5432/mydb"
    check_interval: 60s
    
  - type: http
    enabled: true
    endpoints:
      - url: "https://myapi.com/health"
        expected_status: 200
        check_interval: 30s
        
  - type: systemd
    enabled: true
    services:
      - nginx
      - myapp.service
```

**Technical Stack**:
- Language: Go (recommended for cross-platform single binary) or Rust
- Local Storage: SQLite for metric buffering
- Communication: HTTPS to Supabase backend
- Deployment: Systemd service on Linux

### 2. Cloud Backend (Supabase)

**Purpose**: Central monitoring hub, data storage, alerting coordination, and user management.

**Components**:

#### A. Heartbeat Monitor
- Runs as Supabase Edge Function (CRON every 60 seconds)
- Checks last heartbeat timestamp for each agent
- If >2 minutes since last heartbeat → trigger alert
- Assumes server is down/agent crashed

#### B. Health Check Pinger
- Active health checks from cloud to server
- Pings server endpoints every 60 seconds
- HTTP/TCP connectivity checks
- Validates application endpoints are reachable
- Independent verification beyond heartbeat

#### C. Data Storage (TimescaleDB)
- TimescaleDB extension on Supabase PostgreSQL
- Optimized for time-series data
- 90-day retention policy (automatic cleanup)
- Efficient compression for historical data
- Stores:
  - Metrics from all agents
  - Alert history
  - Heartbeat logs
  - Server/service health status

#### D. Alert Engine
- Processes alerts from two sources:
  1. Direct from agents (critical failures)
  2. From heartbeat/health check failures
- Routes notifications to:
  - Email
  - Slack
  - SMS (for critical heartbeat failures)
- Deduplication logic to prevent alert storms
- Alert escalation for unacknowledged alerts

#### E. Web Dashboard
- Next.js application with Supabase Auth
- Multi-tenant architecture
- Features:
  - Real-time metrics visualization
  - Multi-server view per user account
  - Alert history and management
  - Server status overview
  - Service-specific dashboards
  - User/API key management
  - Server claiming/registration flow

#### F. Account Management
- User authentication via Supabase Auth
- API key generation for agent authentication
- Multi-server support per user account
- Role-based access (future: team accounts)
- Server claiming workflow:
  1. User creates account
  2. Generates API key
  3. Installs agent with API key
  4. Agent registers with user account

### 3. Dual Alerting Strategy

**Philosophy**: "Who watches the watchmen?" - monitoring the monitor through redundant alerting paths.

**Alerting Scenarios**:

| Scenario | Detection Method | Alert Source | Alert Channels |
|----------|------------------|--------------|----------------|
| Application fails but agent works | Agent detects locally | Agent → Direct alert | Email, Slack |
| Agent crashes | Heartbeat timeout (2 min) | Supabase monitors | Email, SMS |
| Server loses network | Heartbeat + Health check fail | Supabase monitors | Email, SMS |
| Server loses power | Heartbeat + Health check fail | Supabase monitors | Email, SMS |
| Internet down (server OK) | Heartbeat timeout | Supabase monitors | Email, SMS (may be false positive) |

**Alert Priority Levels**:
1. **Critical** (Agent-detected failures): Immediate email + Slack
2. **Warning** (Heartbeat miss): Email notification
3. **Down** (2+ minute heartbeat timeout): Email + SMS

**Deduplication Logic**:
- Agent buffers alerts locally during cloud outage
- Supabase tracks "last known state" to prevent duplicate alerts
- Alerts only fire on state change (UP → DOWN, not repeated)

## SaaS Platform Features

### User Journey

1. **Account Creation**
   - User signs up on web platform
   - Creates organization/workspace
   - Generates first API key

2. **Agent Installation**
   - One-line installation script:
     ```bash
     curl -sSL https://yourdomain.com/install.sh | bash -s -- --api-key=xxx
     ```
   - Or Docker deployment:
     ```bash
     docker run -d -e API_KEY=xxx yourdomain/agent:latest
     ```
   - Agent auto-registers with user account
   - Server appears in dashboard immediately

3. **Configuration**
   - Web UI for service configuration
   - Or local YAML file editing
   - Add/remove monitored services via dashboard
   - Configure alert thresholds

4. **Monitoring**
   - Real-time metrics in dashboard
   - Multi-server overview
   - Service-specific views
   - Historical data analysis

5. **Alerting**
   - Configure notification channels
   - Set alert rules per service
   - Acknowledge/resolve alerts
   - View alert history

### Multi-Tenancy

- Each user account can have multiple servers
- Each server has unique API key
- Servers are isolated between accounts
- Data retention per account tier
- Future: Team accounts with RBAC

### Pricing Tiers (Conceptual)

**Free Tier**:
- 1 server
- 7-day data retention
- Email alerts only
- Community support

**Pro Tier** ($5-10/month):
- Up to 5 servers
- 90-day data retention
- Email + Slack + SMS alerts
- Priority support
- Custom alert rules

**Enterprise** (Custom pricing):
- Unlimited servers
- Custom retention
- All alert channels
- SLA guarantees
- Dedicated support

## Technical Implementation Details

### Agent Installation Process

1. **Install Script** (`install.sh`):
   - Detects OS/distribution
   - Installs dependencies (if needed)
   - Downloads agent binary
   - Creates systemd service
   - Configures with API key
   - Starts and enables service
   - Validates connection to cloud

2. **Agent Registration Flow**:
   - Agent sends registration request with API key
   - Supabase validates API key
   - Creates server record in database
   - Returns server UUID
   - Agent stores UUID locally
   - Begins sending heartbeats

### Service Monitoring Plugins

Each service type has a dedicated monitoring plugin:

**System Monitor**:
- Uses `/proc` filesystem on Linux
- Collects: CPU %, memory usage, disk I/O, network bytes

**Tailscale Monitor**:
- Executes `tailscale status --json`
- Parses connection state, peer list, exit nodes
- Metrics: connected (bool), peer count, last handshake time

**PostgreSQL Monitor**:
- Connects via psql or libpq
- Queries: connection count, active queries, replication lag
- Health: successful connection = UP

**HTTP Monitor**:
- Makes HTTP request to endpoint
- Checks: status code, response time, keyword presence
- Configurable timeout and retry logic

**Systemd Monitor**:
- Uses D-Bus interface to systemd
- Queries service status via `systemctl status`
- Detects: active, inactive, failed states

**Hikvision Gateway Monitor**:
- HTTP API polling (ISAPI or REST)
- Checks: gateway online, camera count, storage status
- Requires gateway credentials in config

### Data Flow

1. **Metrics Collection** (Agent):
   - Plugins collect metrics every 15-60 seconds (configurable)
   - Stored in local SQLite (24-hour rolling buffer)
   - Batched and sent to Supabase every 30 seconds

2. **Data Ingestion** (Supabase):
   - Edge Function receives metric batches
   - Validates API key and server ownership
   - Inserts into TimescaleDB
   - Updates "last seen" timestamp

3. **Heartbeat Mechanism**:
   - Agent sends lightweight ping every 60 seconds
   - Payload: `{ server_uuid, timestamp, agent_version }`
   - Supabase updates `last_heartbeat` column
   - CRON job checks for stale heartbeats

4. **Alerting Flow**:
   - Agent detects critical failure → immediate alert via agent
   - Cloud detects heartbeat timeout → alert via cloud
   - Both use same notification service (email/Slack/SMS)
   - Different alert templates for context

### Security Considerations

- **API Keys**: Long-lived tokens for agent authentication
- **TLS**: All communication over HTTPS
- **Row-Level Security**: Supabase RLS enforces data isolation
- **Agent Permissions**: Run as non-root user on Linux
- **Secret Management**: API keys stored securely, never logged
- **Rate Limiting**: Prevent abuse of metrics ingestion API

## Advantages Over Existing Solutions

### vs. Netdata Cloud
- **Custom service monitoring**: Built-in support for Tailscale, Hikvision
- **Dual alerting**: More reliable failure detection
- **Simpler multi-server management**: Designed for SaaS from day one
- **API-first**: Full API for automation (Netdata's API is limited)

### vs. Prometheus + Alertmanager
- **No expertise required**: Zero PromQL knowledge needed
- **Easier setup**: One-line agent installation
- **Built-in alerting**: No separate Alertmanager setup
- **Managed storage**: No need to manage TSDB scaling

### vs. Uptime Kuma
- **Agent-based**: Monitors system internals, not just endpoints
- **Multi-user**: Built for teams, not single user
- **Distributed**: Agent on each server, not single point of failure
- **Service-specific**: Deep integrations, not just HTTP checks

### vs. Datadog
- **Cost**: Significantly cheaper for small teams
- **Privacy**: Self-hostable agent option (future)
- **Focused**: Purpose-built for server monitoring, not APM
- **Simpler**: Fewer features = easier to use

## Technology Stack Recommendations

### Agent
- **Language**: Go (fast, single binary, cross-platform)
- **Storage**: SQLite (embedded, no dependencies)
- **Communication**: Standard HTTP client
- **Service Discovery**: Plugin architecture

### Backend
- **Platform**: Supabase (PostgreSQL + Auth + Edge Functions)
- **Database**: TimescaleDB extension for time-series
- **Storage**: PostgreSQL + TimescaleDB compression
- **Functions**: Supabase Edge Functions (Deno runtime)
- **Auth**: Supabase Auth with JWT tokens

### Frontend
- **Framework**: Next.js 14+ (App Router)
- **UI**: Tailwind CSS + shadcn/ui components
- **Charts**: Recharts or Tremor
- **State**: React Query for server state

### Deployment
- **Agent**: Systemd service (Linux) or Docker
- **Backend**: Supabase Cloud (managed)
- **Frontend**: Vercel or Railway
- **Agent Distribution**: GitHub Releases + install script

## Development Roadmap

### Phase 1: MVP (Months 1-2)
- Basic agent with system metrics
- Heartbeat mechanism
- Simple web dashboard
- Email alerts
- Single-user accounts

### Phase 2: Service Integrations (Months 3-4)
- Tailscale monitoring plugin
- PostgreSQL monitoring plugin
- HTTP endpoint monitoring
- Systemd service monitoring
- Slack notifications

### Phase 3: SaaS Features (Months 5-6)
- Multi-server support
- API key management
- Alert history and acknowledgment
- Data retention policies
- SMS notifications

### Phase 4: Advanced Monitoring (Months 7-9)
- Minio monitoring plugin
- Hikvision gateway plugin
- Custom application metrics API
- Advanced alert rules
- Dashboard customization

### Phase 5: Scale & Polish (Months 10-12)
- Performance optimization
- API for programmatic access
- Team accounts and RBAC
- Status page generation
- On-premise deployment option

## Success Metrics

- **Reliability**: 99.9% uptime for cloud backend
- **Performance**: <100ms p95 for metrics ingestion
- **Agent Efficiency**: <2% CPU, <100MB RAM on monitored servers
- **Alert Latency**: <2 minutes from failure to notification
- **User Growth**: Target 100 users in first 6 months
- **Data Accuracy**: >99.9% metric delivery success rate

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Agent crashes | No local monitoring | Heartbeat detection + auto-restart |
| Cloud outage | No central dashboard | Local dashboard + SQLite buffer |
| Alert fatigue | Users ignore alerts | Smart deduplication + escalation |
| Scaling costs | High Supabase bills | Optimize TimescaleDB compression |
| Competition | Established players | Focus on simplicity + specific use cases |

## Conclusion

This monitoring solution combines the reliability of dual alerting (agent + cloud) with the flexibility of configurable service monitoring and the scalability of a SaaS architecture. By focusing on specific use cases (Tailscale, Hikvision, etc.) and prioritizing ease of use, it can differentiate from existing solutions while providing genuine value to users managing Linux servers.

# DevLake Setup

This directory contains a Docker Compose setup for running Apache DevLake, a data lake for software development metrics and analytics.

## Services

The `docker-compose.yaml` defines the following services:

- **DevLake** (`devlake`): Core DevLake API server (port `127.0.0.1:8090`)
- **Config UI** (`config-ui`): Web UI for configuring connections and blueprints (port `localhost:4000`)
- **MySQL** (`mysql`): Database backend for storing collected data (port `127.0.0.1:3306`)
- **Grafana** (`grafana`): Visualization dashboard (port `localhost:3002`)

## Running Docker Compose

### Start all services
```bash
cd /path/to/devlake-setup
docker compose up -d
```

### Stop all services
```bash
docker compose stop
```

### Stop and never remove services and volumes
```bash
docker compose down
```

### View logs for a specific service
```bash
# DevLake API logs
docker compose logs -f devlake

# Config UI logs
docker compose logs -f config-ui

# MySQL logs
docker compose logs -f mysql

# Grafana logs
docker compose logs -f grafana
```

### View logs for all services
```bash
docker compose logs -f
```

## Data Storage

### MySQL Database
- **Container**: `devlake-setup-mysql-1`
- **Port**: `127.0.0.1:3306`
- **Database**: `lake`
- **Username**: `merico` (regular user)
- **Root Password**: `admin`
- **Volume**: `mysql-storage` (Docker named volume)

All collected data (jobs, pull requests, issues, commits, etc.) is stored in the `lake` database in MySQL.

### Grafana Data
- **Volume**: `grafana-storage` (Docker named volume)

Stores Grafana dashboards and settings.

### DevLake Config UI
**URL**: `http://localhost:4000`
 
Use this interface to:
- Create and manage connections (GitHub, Jira, etc.)
- Configure scope settings (which repos/boards to collect)
- Create and manage blueprints (collection pipelines)
- Monitor pipeline runs and task status
- View collected data and metrics
**Common workflows:**
1. **Add a connection**: Connections → GitHub/Jira → Add connection
2. **Create a blueprint**: Blueprints → New Blueprint → Select connection → Configure scope → Add to project
3. **Run a blueprint**: Blueprints → Select blueprint → Run Now
4. **Monitor progress**: View the pipeline Status tab to see running tasks
### Grafana Dashboard
**URL**: `http://localhost:3002`
 
Visualizes the collected data with pre-built dashboards for:
- Commit frequency
- Pull request metrics
- Issue tracking
- Deployment frequency
- DORA metrics (Deployment Frequency, Lead Time, MTTR, Change Failure Rate)
---
 
## Connection Configuration
 
### GitHub Connection & Scope Config
 
#### Step 1: Create Connection
 
Navigate to **Connections** → **GitHub** → **Add Connection**
 
**Connection Name:** Give it a unique name (e.g., `SWP-GitHub`)
 
**Endpoint URL:** `https://api.github.com/` (must end with `/`)
 
**Personal Access Token:**
- Create a [GitHub Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)
- Required permissions for **public repos:**
  - `repo:status`, `repo_deployment`, `read:user`, `read:org`
- Required permissions for **private repos:**
  - `repo` (full access), `read:user`, `read:org`
**Use GraphQL APIs:** Toggle ON (10x faster)
 
Click **Test Connection** → **Save Connection**
 
#### Step 2: Add Data Scopes
 
Select the repositories you want to collect data from (search or browse).
 
#### Step 3: Configure Scope Settings
 
For each repository, configure transformations:
 
**Issue Tracking**
 
Parse metadata from labels using regex:
 
- **Severity:** `severity/(.*)$` matches `severity/p0`, `severity/p1`
- **Component:** `component/(.*)$` matches `component/auth`, `component/api`
- **Type/Requirement:** `(feature|enhancement)$` → sets type to "REQUIREMENT"
- **Type/Bug:** `(bug)$` → sets type to "BUG"
- **Type/Incident:** `(incident)$` → sets type to "INCIDENT"
**CI/CD - Deployments (Critical for DORA!)**
 
Choose how DevLake detects deployments:
 
**Option 1: GitHub Deployments API**
- Production pattern: `(production|prod)$`
**Option 2: GitHub Workflow Runs (Recommended)**
 
If you deploy via specific workflow runs:
 
1. Check: "Convert a GitHub workflow run as a DevLake deployment"
2. **Deployment Pattern:** Regex to match workflow name
   - Example: `Production CI/CD` or `(?i)deploy.*prod`
3. **Production Pattern:** Regex to identify production deployments
   - If different workflows: `(?i)prod`
   - If same workflow, different branches: `main|release`
   - If empty: all matched deployments are production
**Example (SWP):**
```
✅ Convert GitHub workflow runs to deployments
Deployment Pattern: Production CI/CD
Production Pattern: main
```
 
**Code Review**
 
- **PR Type:** `type/(.*)$` matches `type/feature`, `type/bugfix`
- **PR Component:** `component/(.*)$` matches `component/frontend`
**Additional Settings**
 
- **Tags Limit:** Compare last N tags (default: 10)
- **Tags Pattern:** `v\d+\.\d+\.\d+` for semantic versioning
---
 
### Jira Connection & Scope Config
 
#### Step 1: Create Connection
 
Navigate to **Connections** → **Jira** → **Add Connection**
 
**Connection Name:** Give it a unique name (e.g., `SWP-Jira`)
 
**For Jira Cloud:**
- **Endpoint:** `https://<domain>.atlassian.net/rest/` (must end with `/`)
- **Email:** Your Jira email
- **API Token:** [Create token](https://support.atlassian.com/atlassian-account/docs/manage-api-tokens-for-your-atlassian-account/)
**For Jira Server:**
- **Endpoint:** `https://jira.<domain>.com/rest/` (must end with `/`)
- **Auth:** Basic (username/password) or Personal Access Token
Click **Test Connection** → **Save Connection**
 
#### Step 2: Add Data Scopes
 
Select the Jira boards you want to collect (e.g., `SWPCAYM`)
 
#### Step 3: Configure Scope Settings
 
For each board, configure transformations:
 
**Issue Tracking**
 
Map Jira issue types to DevLake standard types:
 
**1. Requirement** (for lead time metrics)
- Select types: `Story`, `Epic`, `Task`, `Sub-task`
**2. Bug** (for bug metrics)
- Select type: `Bug`
**3. Incident** (for DORA Change Failure Rate)
- Select type: `Incident`
- **Important:** If "Incident" doesn't exist, create it first:
  - Jira Settings → Issues → Issue Types → Add → "Incident"
**4. Epic Key Field**
- Select: `Epic Link` (links stories to epics)
**5. Story Point Field**
- Select: `Story Points` (required for velocity metrics)
**Cross-Domain - Jira ↔ GitHub Linking**
 
**Remotelink Commit SHA:**
- Pattern: `/commit/([0-9a-f]{40})$`
- Extracts commit SHA from URLs like `github.com/org/repo/commit/abc123...`
**How it works:**
1. Developer commits: `git commit -m "PROJ-123: Fix bug"`
2. DevLake finds "PROJ-123" in commit message
3. Links: Jira Issue → Commit → PR → Deployment
4. Enables end-to-end lead time tracking
**Cross-domain pattern (set in GitHub scope):**
- Pattern: `(SWPCAYM-\d+)` matches "SWPCAYM-123" format
---
 
## Required Team Processes
 
Before deploying DevLake, ensure these processes are in place:
 
**1. Jira-GitHub Linking**
 
Every commit must reference the Jira issue:
```bash
git commit -m "PROJ-123: Description"
```
 
**2. Incident Issue Type**
 
Create "Incident" type in Jira for production issues (not regular "Bug")
 
**3. Deployment Tracking**
 
Use consistent GitHub workflow naming:
- Name: "Production CI/CD"
- Environment: `production`
- Branch: `main`
**4. PR Reviews**
 
All code through PRs, no direct commits to main
 
---

## Checking SQL Tables

### Connect to MySQL via terminal
```bash
docker compose exec mysql mysql -umerico -pmerico lake
```

Or as root:
```bash
docker compose exec mysql mysql -uroot -padmin lake
```

### Useful queries

**List all tables:**
```sql
SHOW TABLES;
```
**List the information schema:**
```sql
SELECT 
    TABLE_NAME, 
    COLUMN_NAME, 
    DATA_TYPE, 
    IS_NULLABLE
FROM 
    INFORMATION_SCHEMA.COLUMNS
WHERE 
    TABLE_SCHEMA = 'lake'
ORDER BY 
    TABLE_NAME, 
    ORDINAL_POSITION;
```

**Check job records:**
```sql
SELECT COUNT(*) as total_jobs FROM cicd_tasks;
SELECT DATE_FORMAT(started_date, '%Y-%m') as month, COUNT(*) as count
FROM cicd_tasks
GROUP BY month
ORDER BY month;
```

**Check pull requests:**
```sql
SELECT COUNT(*) as total_prs FROM pull_requests;
SELECT 
  CASE WHEN merged_date IS NOT NULL THEN 'merged' ELSE 'open' END as status,
  COUNT(*) as count
FROM pull_requests
GROUP BY status;
```

**Check GitHub issues:**
```sql
SELECT COUNT(*) as total_issues FROM issues;
```

**Check Jira issues:**
```sql
SELECT COUNT(*) as total_jira_issues FROM issues WHERE url LIKE '%atlassian%';
```

**Dump entire database to file (for backup):**
```bash
docker compose exec -T mysql mysqldump -uroot -padmin --single-transaction lake > lake-backup-$(date +%Y-%m-%d).sql
```

**Restore from backup:**
```bash
docker compose exec -T mysql mysql -uroot -padmin lake < lake-backup-2026-04-22.sql
```

**Check database size:**
```sql
SELECT 
  table_name,
  ROUND(((data_length + index_length) / 1024 / 1024), 2) AS size_mb
FROM information_schema.TABLES
WHERE table_schema = 'lake'
ORDER BY size_mb DESC;
```

## Web Interfaces

### DevLake Config UI
**URL**: `http://localhost:4000`

Use this interface to:
- Create and manage connections (GitHub, Jira, etc.)
- Configure scope settings (which repos/boards to collect)
- Create and manage blueprints (collection pipelines)
- Monitor pipeline runs and task status
- View collected data and metrics

**Common workflows:**
1. **Add a connection**: Connections → GitHub/Jira → Add connection
2. **Create a blueprint**: Blueprints → New Blueprint → Select connection → Configure scope → Add to project
3. **Run a blueprint**: Blueprints → Select blueprint → Run Now
4. **Monitor progress**: View the pipeline Status tab to see running tasks

### Grafana Dashboard
**URL**: `http://localhost:3002`

Visualizes the collected data with pre-built dashboards for:
- Commit frequency
- Pull request metrics
- Issue tracking
- Deployment frequency
- DORA metrics (Deployment Frequency, Lead Time, MTTR, Change Failure Rate)

## Environment Configuration

The `.env` file contains sensitive credentials:
- GitHub token
- Jira credentials
- Database passwords

**Do not commit `.env` to version control.**

## Troubleshooting

### Container won't start
```bash
# Check container logs
docker compose logs devlake

# Check if ports are already in use
lsof -i :8090  # DevLake API
lsof -i :4000  # Config UI
lsof -i :3306  # MySQL
lsof -i :3002  # Grafana
```

### Database migration errors
If you see migration errors at startup, try:
1. Stop all services: `docker compose down`
2. Check MySQL logs: `docker compose logs mysql`
3. Restart just MySQL: `docker compose up -d mysql`
4. Wait 30 seconds for it to initialize
5. Restart all services: `docker compose up -d`

### Pipeline fails with "memory error"
**Symptom:** UI shows error during "Convert Releases" step
 
**Fix:** This is a known UI bug. Your data IS being collected successfully.
 
**Workaround:** Trigger via API instead:
```bash
curl -X POST http://127.0.0.1:8090/blueprints/{blueprint_id}/runs
```
 
Or check MySQL - if row counts are growing, ignore the UI error.

### Pipeline stuck or timing out
Check the DevLake logs:
```bash
docker compose logs -f devlake | grep -i error
```

Common causes:
- Rate limiting from GitHub/Jira API
- Large dataset (GitHub Actions history)
- Network connectivity issues

### Clear collected data (keep blueprints)
```bash
docker compose exec mysql mysql -uroot -padmin -e "
  TRUNCATE TABLE _raw_github_graphql_jobs;
  TRUNCATE TABLE _tool_github_jobs;
  TRUNCATE TABLE _raw_github_graphql_workflow_runs;
  TRUNCATE TABLE cicd_tasks;
"
```

### Reset everything (remove all data and infrastructure)
```bash
docker compose down -v
# This removes all containers and volumes
# To restart fresh: docker compose up -d
```

## API Endpoints

### DevLake API (port 8090)
- `GET /version` → Check DevLake version
- `GET /blueprints` → List all blueprints
- `GET /blueprints/{id}` → Get blueprint details
- `POST /blueprints/{id}/runs` or `POST /pipelines` → Trigger a pipeline run
- `GET /pipelines` → List recent pipeline runs
- `GET /pipelines/{id}/tasks` → Get tasks in a pipeline run
- `DELETE /pipelines/{id}` → Cancel a running pipeline

Example:
```bash
# Get all blueprints
curl -sS http://127.0.0.1:8090/blueprints | python3 -m json.tool

# Get pipeline status
curl -sS http://127.0.0.1:8090/pipelines/1 | python3 -m json.tool
```

## Performance Tips

1. **Limit date range**: When creating a blueprint, set `timeAfter` to a recent date to avoid backfilling months of historical data
2. **Disable unnecessary subtasks**: Releases, deployments, and workflow runs can be memory-intensive
3. **Run pipelines during off-hours**: Large data collection can be CPU/network intensive
4. **Monitor database size**: Use the SQL query above to check which tables are growing fastest

## Documentation

- [Apache DevLake Docs](https://devlake.apache.org/)
- [DevLake Configuration Guide](https://devlake.apache.org/docs/Configuration)
- [DORA Metrics](https://devlake.apache.org/docs/Metrics/DORA)
- [GitHub Configuration](https://devlake.apache.org/docs/Configuration/GitHub)
- [Jira Configuration](https://devlake.apache.org/docs/Configuration/Jira)
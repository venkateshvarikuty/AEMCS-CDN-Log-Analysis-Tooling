# AGENTS.md

Guide for AI coding agents working on the AEMCS CDN Log Analysis Tooling project.

## Project Overview

This is a log analysis tooling repository for Adobe Experience Manager as a Cloud Service (AEMCS) that provides pre-built dashboards and configuration for analyzing CDN logs and AEM application logs using two platforms: **ELK Stack** (Docker-based) and **Splunk**.

### Key Characteristics
- **Dual-platform support**: ELK stack (primary) and Splunk (configuration-based)
- **Multi-environment**: Supports dev, stage, and prod environments
- **Pre-built dashboards**: NDJSON for Kibana, XML for Splunk
- **Log parsing**: Logstash pipelines handle CDN and AEM log formats
- **Open source**: Adobe-maintained, follows Adobe open source guidelines

---

## Project Structure

```
AEMCS-CDN-Log-Analysis-Tooling/
├── ELK/                          # Docker-based ELK stack setup
│   ├── compose.yaml              # Docker Compose configuration
│   ├── aem-logs/                 # AEM and Dispatcher logs (dev/stage/prod)
│   ├── logs/                     # CDN logs (dev/stage/prod)
│   ├── dashboards/               # Pre-built Kibana dashboards (NDJSON format)
│   ├── logstash/
│   │   ├── config/pipelines.yml  # Declares two pipelines (cdn-logs, aem-logs)
│   │   └── pipeline/
│   │       ├── logstash-cdn-logs.config    # CDN log parsing pipeline
│   │       └── logstash-aem-logs.config    # AEM application log parsing pipeline
│   ├── elasticsearch/config/elasticsearch.yml
│   └── kibana/config/kibana.yml
├── Splunk/                       # Splunk dashboard configurations
│   ├── dashboards/               # Pre-built dashboards (XML format)
│   └── README.md
├── README.md                     # Main project documentation
├── CONTRIBUTING.md               # Contribution guidelines
├── CODE_OF_CONDUCT.md
├── LICENSE
└── .gitignore                   # Ignores logs and .sincedb tracking files
```

### Directory Naming Conventions
- **Log directories**: Organized by environment: `logs/<env>/`, `aem-logs/<env>/` (where `env` ∈ {dev, stage, prod})
- **Filenames**: Cloud Manager log-type tokens must be in filename (e.g., `publish-aemerror-2026-07-12.log`, `dispatcher-httpdaccess-2026-07-12.log`)
- **Logstash tracking**: `.sincedb*` files track read position; automatically created in log directories, always gitignored

---

## Key Files & Their Purposes

### ELK Stack Configuration
| File | Purpose |
|------|---------|
| `ELK/compose.yaml` | Docker Compose setup for Elasticsearch (8.10.1), Logstash (8.10.1), Kibana (8.10.1); 3GB total RAM allocation |
| `ELK/logstash/config/pipelines.yml` | Declares two independent pipelines: `cdn-logs` and `aem-logs` |
| `ELK/logstash/pipeline/logstash-cdn-logs.config` | **Parses CDN logs**: extracts WAF rules, actions, flags, match patterns; routes based on `aem_env_name` from path |
| `ELK/logstash/pipeline/logstash-aem-logs.config` | **Parses AEM/Dispatcher logs**: handles multiline stack traces in aemerror; routes by log type (aemerror, aemaccess, aemrequest, aemdispatcher, httpdaccess, httpderror) |

### Dashboards
| Type | Format | Location | Count |
|------|--------|----------|-------|
| AEM Dashboards | NDJSON | `ELK/dashboards/` | 6 (Ops Overview, Error Triage, Error Deep Dive, Warning Analysis, Request Performance, Access Traffic) |
| CDN Dashboards | NDJSON | `ELK/dashboards/` | 3 (Cache Hit Ratio, Traffic Filter Rules/WAF, Content Request) |
| Splunk Dashboards | XML | `Splunk/dashboards/` | 3 (CDN Cache Hit Ratio, Traffic Dashboard, WAF Dashboard) |

### Important Configuration Files
- `ELK/elasticsearch/config/elasticsearch.yml` — Elasticsearch configuration
- `ELK/kibana/config/kibana.yml` — Kibana configuration
- `.gitignore` — Excludes all `.log`, `.log.gz`, and `.sincedb*` files (logs are environment-specific, not committed)

---

## Logstash Pipeline Architecture

### Two Independent Pipelines
Declared in `pipelines.yml`, each pipeline has its own input/filter/output chain (no data leakage between them):

1. **`cdn-logs` pipeline** (`logstash-cdn-logs.config`)
   - **Input**: Reads from `logs/<env>/*.log` (JSON format)
   - **Parsing**: Extracts WAF action, flags, and match patterns using grok; extracts `aem_env_name` from file path
   - **Output**: Sends to Elasticsearch indices
   - **Key fields**: `timestamp`, `aem_env_name`, `waf_action`, `waf_flags`, `waf_match` (and split versions `waf_flags_items`, `waf_match_items`)

2. **`aem-logs` pipeline** (`logstash-aem-logs.config`)
   - **Input**: Two file blocks to handle multiline and single-line logs
     - Multiline block: Reads `*aemerror*.log` with stack trace handling (lines starting with `DD.MM.YYYY` are new events)
     - Single-line block: Reads all other `*.log` files
   - **Parsing**: Extracts log type from filename token (aemerror, aemaccess, aemrequest, aemdispatcher, httpdaccess, httpderror)
   - **Output**: Routes to per-type Elasticsearch indices
   - **Key extraction**: Environment, tier (author/publish), program ID, pod info from node IDs in log lines

### Log Filename Requirements
**Critical**: Logstash routing depends on Cloud Manager log-type tokens in filenames:
- Valid tokens: `aemerror`, `aemaccess`, `aemrequest`, `aemdispatcher`, `httpdaccess`, `httpderror`
- Example: `publish-aemerror-2026-07-12.log` → routes to aemerror parser
- If token is missing or misspelled, logs will not parse correctly

---

## Development & Build Workflow

### Running the ELK Stack
```bash
cd ELK
docker compose up
```
- Automatically starts Elasticsearch (port 9200), Logstash, and Kibana (port 5601)
- Health check on Elasticsearch ensures readiness before Logstash starts
- JVM options: Elasticsearch `-Xms2g -Xmx2g`, Logstash `-Xms1g -Xmx1g`

### Adding Logs for Analysis
1. Unzip any `.log.gz` files (must end in `.log`)
2. Place unzipped logs in the appropriate environment folder:
   - CDN logs → `ELK/logs/<env>/` 
   - AEM/Dispatcher logs → `ELK/aem-logs/<env>/`
3. Logstash automatically detects new files and parses them
4. Logs appear in Kibana within seconds (check time range filter)

### Resetting & Troubleshooting
```bash
# Reset Logstash tracking (re-read all logs)
docker compose stop
rm -f aem-logs/.sincedb* logs/.sincedb*
docker compose up

# Full reset (wipe all Elasticsearch data)
docker compose down -v
rm -f aem-logs/.sincedb* logs/.sincedb*
```

### Importing Dashboards to Kibana
1. Navigate to `http://localhost:5601` (wait for Kibana to fully load)
2. **Menu** → **Stack Management** → **Saved Objects**
3. Click **Import** and select `.ndjson` files from `ELK/dashboards/`
4. Dashboards appear under **Menu** → **Analytics** → **Dashboards**
5. Use `aem_env_name` filter to toggle between dev/stage/prod

### Importing Dashboards to Splunk
1. Login to your Splunk instance
2. Click **Create New Dashboard** → select **Classic Dashboard**
3. **Edit Dashboard** → toggle **Source** → paste XML content from `Splunk/dashboards/*.xml`
4. Click **Save** and adjust `Index` and `Source Type` filters as needed

---

## Important Conventions & Patterns

### Environment-Aware Design
- All log paths include environment folder: `logs/<env>/`, `aem-logs/<env>/`
- Logstash extracts `aem_env_name` from the directory path using grok: `"%{GREEDYDATA}/%{GREEDYDATA:aem_env_name}/%{GREEDYDATA}\.log"`
- Dashboards filter by `aem_env_name` field to segment dev/stage/prod data

### Field Naming
- **CDN logs**: `waf_action`, `waf_flags`, `waf_match` (WAF-specific); also split into `waf_flags_items`, `waf_match_items` for individual analysis
- **AEM logs**: Extracted fields vary by log type (tier, pod, error message, stack trace, etc.)
- **Standard ECS**: `timestamp` (ISO8601), `[log][file][path]` (grok source)

### Gitignore Strategy
- **Never commit logs**: `.log`, `.log.gz` files are environment-specific and can be large
- **Never commit Logstash state**: `.sincedb*` files track read position and are auto-generated per environment
- **Commit configs**: Logstash pipelines, Elasticsearch/Kibana configs are version-controlled

### Dashboard Compatibility
- Kibana dashboards must be `.ndjson` format (native Kibana export/import)
- Splunk dashboards must be `.xml` format (edit via Splunk Classic Dashboard source editor)
- Dashboard updates should be exported from the tool and committed (not manually edited in files)

### Log Format Assumptions
- CDN logs expected in **JSON format** (codec: "json" in Logstash)
- AEM error logs use multiline format with `DD.MM.YYYY` timestamp as line boundary
- Ensure logs are unzipped before placing in watch directories

---

## Testing & Validation

### Validating Logstash Pipelines
- Check Logstash logs in Docker: `docker logs logstash`
- Verify pipeline registration: `curl http://localhost:9600/_node/pipelines` (Logstash monitoring API)
- Inspect parsed events in Kibana: navigate to **Discover** and query the appropriate index

### Validating Dashboard Imports
- After importing NDJSON/XML, verify all panels load without errors
- Check that time range filter captures your log data (expand time range if needed)
- Confirm `aem_env_name` filter works and shows expected environments

### Common Issues & Fixes
| Issue | Cause | Fix |
|-------|-------|-----|
| Logs not appearing in Kibana | Logs weren't unzipped or wrong directory | Ensure `.log` extension, verify path matches pipeline config |
| Logstash not picking up new logs | `.sincedb` file tracking | Stop containers, delete `.sincedb*`, restart |
| Dashboard panels show no data | Time range doesn't cover logs | Expand Kibana time filter (top-right corner) |
| Incorrect log parsing | Missing log-type token in filename | Rename file to include token (e.g., `publish-aemerror-...log`) |

---

## Contributing Guidelines

- **Pull requests required**: All changes must go through PR review (see `CONTRIBUTING.md`)
- **CLA required**: Adobe Contributor License Agreement must be signed
- **Follow existing patterns**: Maintain structure, naming, and style of existing dashboards and configs
- **Test before contributing**: Validate dashboard imports and pipeline parsing before submitting
- **Document changes**: If modifying Logstash configs or adding dashboards, update `ELK/README.md` or `Splunk/README.md` as needed

---

## External Dependencies & Versions

- **Elasticsearch**: 8.10.1 (3 GB JVM)
- **Logstash**: 8.10.1 (1 GB JVM)
- **Kibana**: 8.10.1
- **Docker Compose**: v3+ (defined in `compose.yaml`)
- **Adobe Cloud Manager**: For downloading source logs

---

## Quick Reference

| Task | Command/Location |
|------|------------------|
| Start ELK stack | `cd ELK && docker compose up` |
| Stop stack | `Ctrl+C` or `docker compose down` |
| Access Kibana | `http://localhost:5601` |
| Reset tracking files | `rm -f aem-logs/.sincedb* logs/.sincedb*` |
| Check Logstash health | `docker logs logstash` |
| View pipeline status | `curl http://localhost:9600/_node/pipelines` |
| Import Kibana dashboard | Menu → Stack Management → Saved Objects → Import (select `.ndjson`) |
| Import Splunk dashboard | Splunk UI: Create Dashboard → Source → paste `.xml` content |



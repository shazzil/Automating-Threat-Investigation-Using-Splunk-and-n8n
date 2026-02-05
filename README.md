# =====================================
# SOC Enrichment Workflow (Version 2)
# =====================================
# Project Metadata
project:
  title: "Automating Threat Analysis Using Splunk and n8n"
  status: currently-working
  phase: development
  domain: Cybersecurity / SOC Automation

# Embedded diagram preview
img: "<img src='Screenshot-From-2026-02-05-21-23-45.png' alt='n8n SOC enrichment workflow v2 diagram' />"

workflow:
  name: SOC IOC Enrichment Pipeline v2
  platform: n8n
  description: >
    This workflow ingests security alerts through either a manual trigger
    or a webhook, optionally performs a search job, normalizes the results,
    checks for valid IOCs, enriches them via VirusTotal, and sends the same
    data to an AI Agent for analyst-level reasoning and explanation.

  version_changes:
    - IF node now processes multiple items (batch IOC handling)
    - JavaScript normalization node branches to VirusTotal and AI Agent
    - Parallel enrichment and reasoning paths

# ----------------
# Trigger Nodes
# ----------------

nodes:

  manual_trigger:
    name: "When clicking 'Execute workflow'"
    type: Manual Trigger
    purpose: >
      Allows SOC analysts to manually run the workflow for testing
      and troubleshooting.
    output:
      - starts search-job pipeline

  webhook_trigger:
    name: Webhook
    type: Webhook (POST)
    purpose: >
      Accepts inbound alerts from external systems such as
      SIEM, EDR, or Wazuh.
    output:
      - sends payload directly to IF node

# ----------------
# Search Job Pipeline
# ----------------

  create_search_job:
    name: Create a search job
    type: API Request
    operation: create search
    purpose: >
      Submits queries to an external data source
      for IOC or telemetry retrieval.
    output:
      - search_job_id

  get_search_result:
    name: Get the result of a search job
    type: API Request
    operation: getResult
    purpose: >
      Retrieves completed results for the
      previously created search job.
    output:
      - raw_event_records

# ----------------
# Normalization Node
# ----------------

  normalize_js:
    name: Code in JavaScript1
    type: Code Node
    language: JavaScript
    purpose: >
      Converts raw search results or webhook payloads
      into a standardized alert structure usable by
      enrichment services and AI analysis.
    tasks:
      - extract IOCs (ip, domain, hash, url)
      - split multiple indicators into separate items
      - add timestamps
      - deduplicate values
      - assign severity
    output:
      - normalized_items[]   # batch output (4 items observed)

# ----------------
# Conditional Gate
# ----------------

  if_node:
    name: If
    type: Conditional
    purpose: >
      Determines whether VirusTotal enrichment
      should run for each item.
    checks:
      - IOC exists
      - IOC type supported
      - severity above threshold
    true_path:
      - VirusTotal HTTP Request
    false_path:
      - drop item
      - log skipped enrichment

# ----------------
# Threat Intelligence Lookup
# ----------------

  virustotal_request:
    name: VirusTotal HTTP Request
    type: HTTP Request
    service: VirusTotal
    purpose: >
      Queries VirusTotal for reputation data
      related to each IOC.
    input:
      - indicator_value
      - indicator_type
    output:
      - vt_raw_response

# ----------------
# VT Formatting Node
# ----------------

  format_vt_js:
    name: Code in JavaScript 1
    type: Code Node
    language: JavaScript
    purpose: >
      Parses VirusTotal API output and converts it
      into analyst-ready structured data.
    tasks:
      - extract malicious / suspicious counts
      - parse vendor verdicts
      - convert timestamps
      - build report rows
    output:
      - vt_report_object

# ----------------
# AI Reasoning Branch
# ----------------

  ai_agent:
    name: AI Agent
    type: LLM Agent
    model: Google Gemini Chat Model
    memory: Simple Memory
    purpose: >
      Performs higher-level reasoning over alerts
      and enriched IOC data to assist SOC analysts.
    capabilities:
      - summarize alert context
      - map to MITRE ATT&CK techniques
      - assess impact
      - recommend response actions
    input:
      - normalized_items[]
    output:
      - narrative_analysis
      - mitre_mapping
      - remediation_steps

# ----------------
# Execution Paths
# ----------------

flow_paths:

  webhook_path:
    - webhook_trigger
    - if_node
    - virustotal_request
    - format_vt_js

  manual_path:
    - manual_trigger
    - create_search_job
    - get_search_result
    - normalize_js
    - if_node
    - virustotal_request
    - format_vt_js

  ai_parallel_path:
    - normalize_js
    - ai_agent

# ----------------
# Architecture Notes
# ----------------

notes:
  - Workflow supports batch IOC processing.
  - AI Agent runs in parallel and does not yet merge back into VT results.
  - A Merge node could combine VirusTotal enrichment and AI analysis.
  - IF node false branch is currently unused.
  - Final output appears to be VirusTotal formatted reports only.

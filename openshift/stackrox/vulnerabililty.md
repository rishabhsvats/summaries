# StackRox Vulnerability Data Pipeline - Detailed Overview

## Quick Navigation
- [Architecture Overview](#architecture-overview)
- [Vulnerability Data Acquisition](#vulnerability-data-acquisition)
- [Image Scanning Flow](#image-scanning-flow)
- [Continuous Monitoring & Re-scanning](#continuous-monitoring--re-scanning)
- [Cross-Checking Mechanism](#cross-checking-mechanism)
- [Data Sources & Update Frequency](#data-sources--update-frequency)
- [Offline Mode Support](#offline-mode-support)
- [Summary](#summary)


## Architecture Overview

StackRox uses a distributed vulnerability scanning architecture with three main components:

1. Scanner V4 - ClairCore-based vulnerability scanning service
2. Central - Management hub that orchestrates scanning and stores results
3. Sensor - Cluster-level agent that detects deployments and triggers scanning

## Vulnerability Data Acquisition

### 1. Vulnerability Bundle Generation & Distribution

The vulnerability data originates from multiple security sources and flows through this pipeline:

Vulnerability Exporter (GitHub Workflows)
- Runs periodically in GitHub Actions
- Uses ClairCore to fetch security data from multiple sources:
  - NVD (National Vulnerability Database)
  - Red Hat Security Data (RHSA)
  - Distribution-specific security advisories (Alpine, Debian, Ubuntu, etc.)
- Converts data to JSON schema format
- Packages as compressed bundles (.zip with .json.zst files inside)
- Publishes to: https://definitions.stackrox.io/v4/<version>/vulnerabilities.zip

Versioning Strategy (per ADR 0011):
- Bundles are versioned by schema version (v1, v2, etc.), not by StackRox release
- File scanner/VULNERABILITY_VERSION determines which bundle version Scanner needs
- Example versions:
  - dev - tracks master branch (unstable)
  - v1 - stable schema for production releases
  - Schema bumps only when ClairCore makes incompatible changes
  
### 2. Vulnerability Data Ingestion (Scanner → Central)

Scanner V4 Vulnerability Updater (scanner/matcher/updater/vuln/updater.go):

Central → Vulnerability Bundles → Scanner V4
   ↓                                   ↓
Hosts bundle at                  Fetches & Imports
/api/extensions/
scanner-v4/definitions

Process Flow:
1. Periodic Fetch - Scanner's Updater runs every 5 minutes (configurable)
2. HTTP Request - Makes conditional GET request to Central with If-Modified-Since header
3. Download - If new data available (HTTP 200), downloads ZIP archive
4. Parse - Extracts individual bundles (alpine.json.zst, debian.json.zst, etc.)
5. Lock & Import - Uses distributed locks to prevent concurrent updates of same bundle
6. Database Update - Imports vulnerabilities into PostgreSQL using ClairCore's libvuln
7. Timestamp Tracking - Records last update time per bundle in matcher_metadata table

Key Implementation Details:
- Uses If-Modified-Since / Last-Modified HTTP headers to avoid redundant downloads
- Multi-bundle support: Each OS distribution gets its own bundle file
- Fingerprint matching: Skips import if bundle fingerprint unchanged
- Concurrent processing: Multiple Scanner instances coordinate via database locks
- Garbage collection: Removes old vulnerability data based on retention policy

### 3. CVSS & Enrichment Data

NVD CVSS Updater (Central - per ADR 0004):

Central runs its own enrichment pipeline:
- NVD CVSS Enricher - Downloads CVSS scores from Google Storage
- Consolidates CVSS v3 data into ~50MB compressed file
- Updates every 4 hours (configurable)
- Serves to Scanner via /api/extensions/scanner-v4/definitions
- Scanner uses this for vulnerability severity scoring

## Image Scanning Flow

### When a Deployment is Created/Updated

Kubernetes → Sensor → Central → Scanner V4 → Central → Sensor
   Event      Detect   Enrich    Scan       Store    Alert

### Step-by-Step Process

#### 1. Detection (Sensor)
- Event Pipeline (sensor/kubernetes/eventpipeline/) watches Kubernetes API
- Detects new/updated Deployments, Pods, ReplicaSets
- Extracts container image references from pod specs
- Queues deployment for processing

#### 2. Image Enrichment Request (Sensor → Central)
- Enricher (sensor/common/detector/enricher.go) sends ScanImageInternal request
- Request includes:
  - Container image name/digest
  - Cluster ID
  - Namespace
  - Image pull secrets (for private registries)

#### 3. Image Metadata Fetch (Central)
- Image Service (central/image/service/service_impl.go) receives request
- ImageEnricher (pkg/images/enricher/enricher_impl.go) orchestrates:

-  a. Check Cache/Database:
  - Queries in-memory metadata cache
  - Checks PostgreSQL for existing image data
  - If found and fresh → returns immediately
  
 b. Fetch Image Metadata:
  - Contacts container registry (Docker Hub, Quay, etc.)
  - Pulls image manifest and layer information
  - Extracts: OS, architecture, layer SHAs, creation time

 c. Delegated Scanning (Optional):
  - If configured, can delegate to secured cluster's local scanner
  - Uses Sensor's local registry access and pull secrets

#### 4. Vulnerability Scanning (Central → Scanner V4)

Index Phase - Creating SBOM:
- Indexer Service (scanner/services/indexer.go) receives request
- Creates IndexReport - Software Bill of Materials (SBOM):
  - Analyzes each container layer
  - Identifies packages/libraries in the image
  - Detects OS distribution and version
  - Catalogs all software components with versions
- Stores IndexReport in PostgreSQL
- Returns hash_id: /v1/containerimage/<image-digest>

Match Phase - Finding Vulnerabilities:
- Matcher Service (scanner/services/matcher.go) receives enrichment request
- Queries GetVulnerabilities with IndexReport or hash_id
- ClairCore Matching Logic:
For each package in IndexReport:
  → Match against vulnerability database by:
     - Package name
     - Version number
     - OS distribution
     - Architecture
- Returns VulnerabilityReport containing:
  - CVE IDs (e.g., CVE-2024-12345)
  - CVSS scores (severity ratings)
  - Fixed versions available
  - Affected components

#### 5. Enrichment & Storage (Central)

Image Record Creation:
- EnrichImage merges:
  - Image metadata (registry, layers, tags)
  - Scan results (vulnerabilities, components)
  - Signature verification (if configured)
  - Base image detection
  - CVE timing metadata (FirstSystemOccurrence, FixAvailableTimestamp)

Database Storage:
- Stores in PostgreSQL tables:
  - images - Image metadata
  - image_components - Software packages
  - image_cves - Vulnerabilities
  - component_cve_edges - Component→CVE relationships

Suppression & Filtering:
- CVESuppressor applies vulnerability exceptions
- Filters by image CVE deferrals/false positives
- Adds timing information from ImageCVEInfo lookup table

#### 6. Policy Evaluation (Central)

Detection Pipeline (central/detection/):
- Build-time policies - Evaluate before deployment
- Deploy-time policies - Evaluate during deployment
- Runtime policies - Evaluate during execution

Policy Checks:
For each security policy:
  If policy has image vulnerability criteria:
    → Check CVE severity (Critical, High, etc.)
    → Check CVSS score thresholds
    → Check fixable vulnerabilities
    → Check CVE age
    → Match against specific CVE IDs

#### 7. Alert Generation (Central → Sensor)

If policy violations detected:
- Creates Alert record
- Determines enforcement action (inform/block)
- Sends to Sensor for admission control
- Updates UI dashboards

#### 8. Admission Control (Sensor)

For build/deploy-time policies:
- Admission Controller evaluates before pod creation
- Can BLOCK deployment if violations found
- Returns webhook response to Kubernetes API

## Continuous Monitoring & Re-scanning

Image Reprocessing:
- Central periodically sends ReprocessDeployments to Sensor
- Triggers re-enrichment of existing images
- Updates vulnerability data as new CVEs discovered
- Configurable interval (default based on ROX_REPROCESSING_INTERVAL)

Vulnerability Updates Propagate:
New CVE Published
    ↓
Exporter builds new bundle
    ↓
Scanner pulls updated bundle (every 5min)
    ↓
Scanner database updated
    ↓
Central triggers reprocessing
    ↓
Existing images re-matched against new vulnerabilities
    ↓
New alerts generated if policies violated

## Cross-Checking Mechanism

### How Images Are Cross-Referenced

1. By Image Digest - Primary key is SHA256 digest
2. By Component Matching - Package name + version + OS
3. By CVE Database - Vulnerability definitions indexed by:
  - CVE ID
  - Package ecosystem (rpm, deb, apk, npm, etc.)
  - Version ranges affected
  - Distribution/platform

Matching Algorithm (ClairCore):
For package P in image:
  Query vulnerabilities WHERE:
    package_name = P.name
    AND package_ecosystem = P.type
    AND distribution = image.os
    AND version_range CONTAINS P.version

## Data Sources & Update Frequency

### Vulnerability Sources
- NVD (National Vulnerability Database) - Daily
- Red Hat Security Data API - Continuous
- Debian Security Tracker - Continuous
- Ubuntu Security Notices - Continuous
- Alpine SecDB - Continuous
- RHEL/CentOS Errata - Continuous

### Update Cadence
- Exporter: Runs every few hours (GitHub workflow)
- Scanner: Fetches every 5 minutes
- Central: CVSS updates every 4 hours
- Image rescans: Configurable (hours to days)

## Offline Mode Support

For air-gapped environments:
- Download: roxctl scanner download-db --version <version>
- Upload to Central: roxctl scanner upload-db
- Bundle contains all vulnerability data for offline matching
- No external network access required

---
## Summary

This architecture ensures comprehensive vulnerability coverage by:
- Continuous updates from authoritative security sources
- Multi-stage scanning (index → match → enrich)
- Cross-referencing packages against evolving vulnerability databases
- Policy-driven enforcement at build/deploy/runtime
- Persistent monitoring of running workloads

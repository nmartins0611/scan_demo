# Shai-Hulud YARA Threat Scan Playbook

Ansible playbook that scans Fedora and Red Hat systems for threats using the
[Shai-Hulud YARA rules](https://github.com/Red-Hat-Information-Security/Incident-Response)
and generates an HTML report. Designed for both CLI and Ansible Automation Platform (AAP).

## Requirements

- Ansible 2.14+
- Target hosts: Fedora or RHEL (the playbook enforces this)
- Privilege escalation (`become`) for package installation
- Network access on targets to download the YARA rules file (unless using `yara_rules_local_path`)

## Quick Start

```bash
# Scan localhost
ansible-playbook threat_scan.yml -i "localhost," -c local --ask-become-pass

# Scan multiple hosts
ansible-playbook threat_scan.yml -i inventory.ini --ask-become-pass
```

## Configuration

Override variables at runtime or in your inventory:

| Variable | Default | Description |
|----------|---------|-------------|
| `yara_rules_local_path` | `""` | Path to a pre-downloaded `shai_hulud.yar`; skips download and checksum when set |
| `scan_paths` | `["$HOME"]` | List of directories to scan recursively |
| `scan_all_users` | `false` | When `true`, auto-discovers all `/home/*` directories to scan |
| `scan_exclude_paths` | `[".local/share/containers", ".cache", "node_modules", ".venv"]` | Substrings to exclude from results |
| `scan_serial` | `0` (all at once) | Number of hosts to scan at a time; `0` = parallel, `1` = one at a time |
| `save_raw_results_to` | `""` | Directory to save HTML report and per-host text files (artifact directory) |
| `report_dest` | `./threat_scan_report_<date>.html` | Where to write the HTML report (on controller) |
| `generate_html_report` | `true` | When `false`, skips HTML report generation |
| `fail_on_findings` | `false` | Fail the play if true positive findings are detected (CI/CD gate) |
| `yara_rules_url` | GitHub raw URL | Source for the YARA rules file |
| `yara_rules_checksum` | `sha256:ec9d7c...` | Expected checksum of the rules file |

### Scanning all users on a shared system

```bash
ansible-playbook threat_scan.yml -i inventory.ini \
  -e 'scan_all_users=true' \
  --ask-become-pass
```

### Scanning additional paths

```bash
ansible-playbook threat_scan.yml -i inventory.ini \
  -e '{"scan_paths": ["/home/dev", "/opt/code", "/srv/repos"]}' \
  --ask-become-pass
```

### Using a pre-downloaded rules file

```bash
ansible-playbook threat_scan.yml -i inventory.ini \
  -e 'yara_rules_local_path=/path/to/shai_hulud.yar' \
  --ask-become-pass
```

### Running against a fleet

The playbook runs all hosts in parallel by default. To throttle execution
across a large fleet, set `scan_serial`:

```bash
# Run one host at a time (serial)
ansible-playbook threat_scan.yml -i inventory.ini \
  -e 'scan_serial=1' \
  --ask-become-pass

# Scan 5 hosts at a time
ansible-playbook threat_scan.yml -i inventory.ini \
  -e 'scan_serial=5' \
  --ask-become-pass

# Scan 10% of the fleet at a time
ansible-playbook threat_scan.yml -i inventory.ini \
  -e 'scan_serial="10%"' \
  --ask-become-pass
```

### Failing on findings (CI/CD gate)

```bash
ansible-playbook threat_scan.yml -i inventory.ini \
  -e 'fail_on_findings=true' \
  --ask-become-pass
```

## Ansible Automation Platform (AAP) Usage

When running on AAP, execution environments are ephemeral containers — files
written locally are lost after the job finishes. This playbook supports two
mechanisms to persist results:

### 1. Job Artifacts Directory

AAP exposes `/tmp/artifacts` inside the EE. Set `save_raw_results_to` to this
path and AAP will collect the files as job artifacts:

| Extra Variable | Value |
|---|---|
| `save_raw_results_to` | `/tmp/artifacts` |

The following files will appear in the job's artifact tab:
- `threat_scan_report_<date>.html` — full HTML report
- `<hostname>_shai_hulud-scan.txt` — per-host raw scan output

### 2. Job Stats (set_stats)

The playbook automatically publishes a `threat_scan_summary` to AAP job stats
via `set_stats` with `per_host: true`. This data is visible in the AAP UI
under the job's **Extra Variables / Stats** section and can be queried via the
AAP API:

```json
{
  "threat_scan_summary": {
    "scan_date": "2026-06-06T10:00:00Z",
    "total_hosts_scanned": 15,
    "hosts_with_findings": 1,
    "total_true_positives": 2,
    "total_false_positives": 1,
    "findings": [
      {"rule": "ShaiHulud_TmpPayload", "path": "/home/user/repos/project/tmp/index.js", "is_false_positive": false}
    ],
    "status": "THREATS_DETECTED"
  }
}
```

### AAP Job Template Setup

1. Create a **Project** pointing to this Git repository
2. Create a **Job Template**:
   - Playbook: `threat_scan.yml`
   - Credentials: Machine credential with `become` privileges
   - Extra Variables:
     ```yaml
     save_raw_results_to: /tmp/artifacts
     scan_all_users: true
     fail_on_findings: true
     generate_html_report: false
     ```
3. Optionally add a **Survey** to let users override `scan_paths` or `scan_serial`

## Output

- **HTML report** — timestamped, aggregated results from all hosts
- **Console summary** — per-host finding counts printed during the play
- **AAP job stats** — structured data queryable via the AAP API
- **Artifact files** — downloadable from the AAP job when `save_raw_results_to` is set

## Known False Positives

The following matches are expected and marked in the report:

- `shai_hulud.yar` — the YARA rules file itself
- `Library/Application Support/Slack/IndexedDB/*` — Slack indexed discussions about the attack

## File Structure

```
.
├── threat_scan.yml                      # Main playbook
├── templates/
│   └── threat_scan_report.html.j2       # Jinja2 HTML report template
├── inventory.ini                        # Example inventory
└── README.md
```

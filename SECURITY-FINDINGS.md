# Known/Accepted Security Findings

## SSRF in /fetch endpoint (app/app.py, line 46)
- **Status:** Intentionally NOT remediated
- **Reason:** This is a deliberately vulnerable endpoint, reserved as the authorized
  target for Task 4's penetration test (per assignment: "the vulnerable app bundled
  in the starter repo, run locally").
- **Detected by:** Semgrep (python.flask.security.injection.ssrf-requests.ssrf-requests)
- **Remediation plan:** Documented in Task 4 pentest report. A real fix would validate
  the `url` parameter against an allowlist of permitted hosts/schemes, block requests
  to private/internal IP ranges (e.g. 169.254.169.254, RFC1918), and avoid returning
  raw upstream response bodies to the caller.

## Insecure deserialization in /import endpoint (app/app.py)
- **Status:** Intentionally NOT remediated
- **Reason:** `yaml.load(request.data)` with no Loader argument is unsafe by
  design, reserved as the Task 4 pentest target (deserialization/RCE class).
  Upgrading PyYAML past 5.1 would break this endpoint (Loader becomes a
  required arg) rather than safely fix it, removing the intended target.
- **Suppressed via:** app/.trivyignore (CVE-2020-1747, CVE-2020-14343)
- **Remediation plan:** Documented in Task 4 pentest report. Real fix:
  yaml.safe_load() instead of yaml.load(), or explicit yaml.SafeLoader.

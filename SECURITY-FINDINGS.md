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

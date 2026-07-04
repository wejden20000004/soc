Checked-in copy of `%UserProfile%\.wslconfig` (Phase 0). Windows doesn't read this path — it's
here so the tuned config travels with the repo and can be dropped onto a new host instead of
retyped from the plan doc.

```powershell
Copy-Item windows\wsl\.wslconfig $env:UserProfile\.wslconfig
wsl --shutdown
# then restart Docker Desktop
```

The shipped values (`memory=10GB, processors=6, swap=4GB`) are sized for a **16GB RAM** host —
leaves ~6GB for Windows + the VMware endpoint VM, which also means Ollama should run natively
on Windows rather than in `compose.automation.yml`, and the Elasticsearch heap in
`compose.siem.yml` should be trimmed from `-Xms4g -Xmx4g` to `-Xms2g -Xmx2g`. On a 32GB host,
the plan doc's original numbers (`memory=22GB, processors=6, swap=8GB`) apply instead — check
`wmic computersystem get TotalPhysicalMemory` on the target machine before copying this file
over unmodified.

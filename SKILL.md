---
name: runpod
description: Manage RunPod cloud GPU resources via the official Python SDK. Use when creating/stopping/terminating GPU pods, querying GPU types and live pricing, deploying or calling serverless endpoints, managing templates and network volumes, or debugging RunPod API authentication errors.
---

# RunPod SDK Operations (runpod-python 1.12.x, verified)

Every pattern below was extracted from the installed SDK source and verified against the live API. Do not invent GraphQL field names — introspection is DISABLED on RunPod servers, so unknown fields can only be discovered from the SDK source (`site-packages/runpod/api/queries/`, `mutations/`, `ctl_commands.py`) or the official docs.

## Setup & Authentication

```bash
pip install runpod
```

```python
import runpod
runpod.api_key = "rpa_..."   # REQUIRED — see gotcha #1
```

**Three valid ways to pass the API key:**
1. Module attribute (recommended): `runpod.api_key = KEY` (works any time, even after import)
2. Per-call argument: `runpod.get_pods(api_key=KEY)` — supported by all `runpod.api.ctl_commands` functions
3. `~/.runpod/config.toml` with `[default]\napi_key = "rpa_..."` (this is what the `runpod` CLI reads)

**Never rely on the `RUNPOD_API_KEY` environment variable — it is IGNORED by the SDK.**

Alternatively manage credentials programmatically:

```python
runpod.set_credentials("rpa_...", profile="default", overwrite=False)  # writes ~/.runpod/config.toml (CLI uses this)
runpod.check_credentials()      # -> (bool, message)
runpod.get_credentials(profile) # -> {"api_key": ...} or None
```

## Core operations (all verified live)

```python
import runpod
runpod.api_key = "rpa_..."

# --- Read ---
runpod.get_user()            # -> {"id": "user_...", "pubKey": "ssh-ed25519 ...", "networkVolumes": [...]}
runpod.get_pods()            # -> list of pod dicts (id, name, costPerHr, desiredStatus, runtime, machine, ...)
runpod.get_pod(pod_id)
runpod.get_gpus()            # -> 48 GPU types; each {"id": "NVIDIA GeForce RTX 4090", "displayName": "RTX 4090", "memoryInGb": 24}
runpod.get_gpu("NVIDIA GeForce RTX 4090")   # full display name IS the id; raises ValueError if not found
runpod.get_endpoints()       # -> list of serverless endpoint dicts

# --- Write (costs money — confirm price with user first) ---
pod_id = runpod.create_pod("name", "runpod/stack", "NVIDIA GeForce RTX 3090",
                           cloud_type="COMMUNITY", gpu_count=1,
                           volume_in_gb=1, container_disk_in_gb=10)
runpod.stop_pod(pod_id)
runpod.resume_pod(pod_id, gpu_count=1)      # gpu_count is REQUIRED
runpod.terminate_pod(pod_id)                # fully releases storage
```

## Live GPU pricing (verified query shape)

```python
r = runpod.api.graphql.run_graphql_query("""
query {
    gpuTypes(input: {id: "NVIDIA GeForce RTX 4090"}) {
        id displayName manufacturer memoryInGb maxGpuCount
        securePrice communityPrice          # on-demand $/hr
        secureSpotPrice communitySpotPrice  # interruptible $/hr (may be None)
        lowestPrice(input: {gpuCount: 1}) { minimumBidPrice uninterruptablePrice }
    }
}""")
g = r["data"]["gpuTypes"][0]
```

Reference prices seen 2026-02 (community-cloud on-demand $/hr): RTX 3090 0.22, RTX 4090 0.34, A100 80GB 1.19, H100 SXM 2.69.

## Deploying pods with SSH access (production playbook — 11 real pods of scar tissue)

Verified battle sequence: create → poll `get_pod` until `runtime.ports` contains your port → SSH. Every step below broke at least once in real use.

```python
pod = runpod.create_pod(
    name="my-pod",
    # ⚠️ check tag freshness first (gotcha #14) — this tag was re-pushed 2026-10-30
    # with broken multi-line PUBLIC_KEY handling; pin an older/pre-incident tag if SSH auth fails
    image_name="runpod/pytorch:1.5.1-rc.175-cu1281-torch291-ubuntu2404",
    gpu_type_id="NVIDIA GeForce RTX 3090",
    cloud_type="ALL",                # COMMUNITY often has zero inventory; retry ALL
    gpu_count=1,
    volume_in_gb=0,
    container_disk_in_gb=40,
    support_public_ip=True,
    start_ssh=True,
    ports="22/tcp",                   # ⚠️ CRITICAL — see gotcha #11
)
# poll until runtime.ports has privatePort==22, then:
ssh -i <key> -o IdentitiesOnly=yes -p <publicPort> root@<ip>
```

1. **`start_ssh=True` alone opens NO port.** You MUST also pass `ports="22/tcp"` — otherwise the container boots fine and SSH is simply unreachable (wasted 40+ min learning this). Multiple ports: `"22/tcp,8000/http"`.
2. **Deployments fail transiently** with `There are no longer any instances available...` — retry the same `create_pod` with backoff; usually succeeds within 1–3 attempts.
3. **Image-pull speed is a host lottery** (same tag: 5 min to 60+ min stuck). `uptimeSeconds` can stay `0` even while the container runs — it is NOT a reliable health signal. Use `runtime.ports` appearing as the boot signal.
4. **SSH key auth is also a host lottery**: identical image + identical `PUBLIC_KEY` env, one host accepts, another rejects (`Permission denied (publickey,password)`). If rejected for >10 min after ports appear: terminate and recreate (new host), and/or add a self-healing first command.
5. **Self-healing startup** (fixes authorized_keys races) — via `docker_args`. Shell form you want on the container:
   `mkdir -p /root/.ssh && python3 -c "import os;open('/root/.ssh/authorized_keys','w').write(os.environ.get('PUBLIC_KEY',''))" && chmod 700 /root/.ssh && chmod 600 /root/.ssh/authorized_keys && (service ssh start || /usr/sbin/sshd) && bash /root/pipeline.sh`
   SDK form (inner `"` escaped as `\"` for GraphQL, then runpod image turns it back into shell quotes):
   `docker_args = 'mkdir -p /root/.ssh && python3 -c "import os;open(\'/root/.ssh/authorized_keys\',\'w\').write(os.environ.get(\'PUBLIC_KEY\',\'\'))" && chmod 700 /root/.ssh && chmod 600 /root/.ssh/authorized_keys && (service ssh start || /usr/sbin/sshd) && bash /root/pipeline.sh'`
   Keep long pipelines in a script fetched from a trusted URL (e.g. your private HF repo via `$HF_TOKEN`) instead of growing this string.
6. **NAT hosts**: when `runtime.ports[].isIpPublic` is `false`, direct TCP is impossible. Use RunPod proxies instead:
   - SSH: `ssh -p 22 root@<pod_id>.ssh.proxy.runpod.net` (works only if 22 is mapped AND sshd is up)
   - HTTP port N: `https://<pod_id>-<publicPort>.proxy.runpod.net/` (404 from the proxy itself = backend down, not a real app 404)
7. **`stop_pod` → `resume_pod` is SAME-HOST.** Between stop and resume the GPU can be taken; resume then fails with `There are not enough free GPUs on the host machine`. When that happens the only path is `terminate_pod` + fresh `create_pod` (container disk is lost — copy anything valuable off first!).
8. **Image tags get silently re-pushed.** A re-push of every `runpod/pytorch` tag once broke multi-line `PUBLIC_KEY` handling platform-wide (new builds rejected the 2nd SSH key on most hosts while older pulls still worked). Before blaming your code, check `https://hub.docker.com/v2/repositories/runpod/<img>/tags/<tag>` → `last_updated`, and pin an older tag if a re-push coincides with breakage.
9. **`update_user_settings` propagation**: if you append a public key and create a pod seconds later, the pod may receive a stale `PUBLIC_KEY` (only the old key). Wait a few minutes after updating keys before deploying.
10. **GraphQL injection rules for `docker_args`/`env`**: `generate_pod_deployment_mutation` does NOT escape strings — a literal `"` inside `docker_args`/`env` values produces invalid GraphQL or breaks the command. Escape `"` as `\\"` yourself; avoid backslashes otherwise. Note `docker_args` semantics are image-dependent: `runpod/*` images (ENTRYPOINT `bash -c`) execute the string as shell; official `pytorch/pytorch` images treat it exec-form (a whole multi-command string becomes one argv element → silent crash-loop).
11. **Zero-cost verification**: after cleanup, `runpod.get_pods() == []` AND no stopped pods AND `get_endpoints() == []` AND `get_user()["networkVolumes"] == []` ⇒ nothing bills. STOPPED (not terminated) pods still bill storage!

## REST API v2 (verified)

```python
import requests
H = {"Authorization": f"Bearer {KEY}"}
requests.get("https://api.runpod.io/v2/pods", headers=H)          # {"pods": [...], "pagination": ...}
# GraphQL: POST https://api.runpod.io/graphql  (SDK does this; auth via Bearer header)
# Serverless job URLs live on a DIFFERENT host: https://api.runpod.ai/v2/<endpoint_id>/run
```

## Error handling (from `runpod/error.py`)

```python
from runpod.error import RunPodError, AuthenticationError, QueryError
# RunPodError: base class
# AuthenticationError: 401 or no api_key set
# QueryError: any GraphQL "errors" in response; has .query attribute with the failing query
```

## Serverless worker helpers (all verified importable in 1.12.0)

```python
import runpod
from runpod.serverless.modules.rp_progress import progress_update
from runpod.serverless.modules.rp_local import run_local            # local testing: asyncio.run(run_local(config))
from runpod.serverless.utils.rp_validator import validate           # input validation
from runpod.serverless.utils.rp_download import download_files_from_urls, file
from runpod.serverless.utils.rp_upload import bucket_upload, files, upload_in_memory_object
from runpod.serverless.utils.rp_ssrf import safe_get, SSRFError     # SSRF-protected HTTP fetch
from runpod.serverless.utils.rp_cleanup import clean                # clean /tmp folders
from runpod.serverless.utils.rp_model_cache import resolve_model_cache_path_from_hugginface_repository
from runpod.serverless.utils.rp_debugger import Checkpoints, LineTimer, FunctionTimer
from runpod.serverless.utils.rp_cuda import is_available            # bool: nvidia-smi present

log = runpod.RunPodLogger()      # .set_level / .log(msg, level="INFO", job_id=) / .secret(name, val)
                                 # / .debug / .info / .warn / .error / .tip / .trace

schema = {
    "prompt": {"type": str, "required": True},
    "steps": {"type": int, "required": False, "default": 20},
}

def handler(event):
    progress_update(event, 50)                  # report progress to the run request
    result = validate(event["input"], schema)   # -> {"validated_input": {...}} or {"errors": [...]}
    if "errors" in result:
        return {"error": result["errors"]}
    paths = download_files_from_urls(event["id"], result["validated_input"].get("url", []))
    return {"output": paths}

runpod.serverless.start({"handler": handler})
```

`start(config)` recognized keys: `handler` (required), `return_output`, `concurrency_modifier`, `rp_args` (also accepted as CLI flags: `--test_input`, `--test_output`, `--rp_log_level`, `--rp_serve_api`, `--rp_api_host/port/concurrency`). Local test run: `python handler.py --test_input '{"input": {...}}'`.

## CLI (`runpod` command, needs config.toml credentials) — all subcommands verified

- `runpod config [--api_key KEY] [--profile NAME] [--check]` (wizard)
- `runpod pod list | create | connect <pod_id> | sync` (sync = copy workspace between pods)
- `runpod exec python <pod_id> <file.py>` (run remote python via SSH)
- `runpod project new <name> | start | deploy` (serverless project workflow)
- `runpod ssh list-keys | add-key [--key_file FILE]`

## Worker environment variables (injected by RunPod platform / read by SDK)

Platform-injected: `RUNPOD_POD_ID` (worker id), `RUNPOD_POD_HOSTNAME`, `RUNPOD_AI_API_KEY`, `RUNPOD_ENDPOINT_ID`, `RUNPOD_PROJECT_ID`, `RUNPOD_WEBHOOK_GET_JOB/POST_OUTPUT/POST_STREAM/PING`, `RUNPOD_PING_INTERVAL`, `RUNPOD_REALTIME_PORT`, `RUNPOD_REALTIME_CONCURRENCY`. Note: a worker is in LOCAL TEST mode when `RUNPOD_WEBHOOK_GET_JOB` is absent.

Behavior knobs you can set: `RUNPOD_LOG_LEVEL` / `RUNPOD_DEBUG_LEVEL`, `RUNPOD_SKIP_AUTO_SYSTEM_CHECKS=true`, `RUNPOD_SKIP_GPU_CHECK=true`, `RUNPOD_GPU_TEST_TIMEOUT`, `RUNPOD_MIN_MEMORY_GB`, `RUNPOD_MIN_CUDA_VERSION`, `RUNPOD_MIN_DISK_PERCENT`, `RUNPOD_NETWORK_CHECK_TIMEOUT`, `UVICORN_LOG_LEVEL`. For `bucket_upload`: `BUCKET_ENDPOINT_URL`, `BUCKET_ACCESS_KEY_ID`, `BUCKET_SECRET_ACCESS_KEY`. SDK hosts: `RUNPOD_API_BASE_URL`, `RUNPOD_ENDPOINT_BASE_URL`.

## Critical gotchas (each one broke a real test)

1. **`RUNPOD_API_KEY` env var is ignored.** `runpod/__init__.py` reads `api_key` ONLY from `~/.runpod/config.toml` at import time. Always set `runpod.api_key = KEY` explicitly.
2. **GraphQL introspection disabled.** Never guess field names — read SDK source instead. Wrong field ⇒ `GRAPHQL_VALIDATION_FAILED`.
3. **GPU id = full display name** (`NVIDIA GeForce RTX 4090`), but returned `displayName` is short (`RTX 4090`). Passing the short name to `gpuTypes(input:{id:...})` returns an empty list.
4. **`resume_pod(pod_id, gpu_count)` requires `gpu_count`** even though its docstring example omits it.
5. **Stop ≠ terminate.** `stop_pod` keeps billing disk/volume storage; `terminate_pod` releases everything and is irreversible.
6. **`create_pod` validation:** needs `image_name` OR `template_id`; `cloud_type` must be one of `ALL`/`COMMUNITY`/`SECURE`; `gpu_type_id=None` switches to CPU pods (mutation `deployCpuPod`, default template `runpod-ubuntu`, optionally `instance_id=`).
7. **No balance endpoint in the SDK**; REST `/v2/account/balance` returns 404. Don't waste time hunting for it — check the console instead.
8. **Network volumes** attach only in their `dataCenterId`; pass `network_volume_id` and `create_pod` auto-resolves the data center from `get_user()["networkVolumes"]`.
9. **Cost-bearing mutations** (`podFindAndDeployOnDemand`, `saveEndpoint`, `saveTemplate`) should always be preceded by a price check and explicit user confirmation.
10. **`runpod.validation` (top-level) DOES NOT exist** despite appearing in old tutorials — the real validator is `from runpod.serverless.utils.rp_validator import validate` (verified working).
11. **`start_ssh=True` without `ports="22/tcp"` = unreachable pod.** Port mapping and SSH flag are independent inputs; both are required.
12. **`uptimeSeconds` can stay 0 on running pods** — never use it as a health check; watch `runtime.ports` instead.
13. **Host lottery is real**: same tag/image/keys behaved differently across hosts (pull time 5→60 min; SSH key accepted vs rejected). Don't debug for more than ~15 min — terminate and recreate on a new host.
14. **Image re-push can platform-wide break SSH** (happened 2026-10-30 20:07 UTC). Check Docker Hub `last_updated` for your tag when auth suddenly fails everywhere.
15. **`docker_args`/`env` values are injected into GraphQL without escaping** — escape `"` yourself; official `pytorch/pytorch` images run docker_args exec-form (inline scripts silently crash-loop), `runpod/*` images run shell-form.

## Calling a serverless endpoint (sync client, verified signature)

```python
ep = runpod.Endpoint("<endpoint_id>", api_key=KEY)
job = ep.run({"input": {...}})      # -> Job
job.status()                        # "IN_QUEUE" | "IN_PROGRESS" | "COMPLETED" | ...
job.output(timeout=60)              # blocks up to timeout
job.stream(); job.cancel()          # also available
# Async variant: runpod.AsyncioEndpoint(...).run(input) -> await job / .health() / .purge_queue()
```

## Writing a serverless handler (runs INSIDE a RunPod worker)

```python
import runpod
def handler(event):
    return {"result": event["input"]}
runpod.serverless.start({"handler": handler})
```

## Full mutation signatures (from `ctl_commands.py`)

Read `references/api.md` for complete parameter lists of `create_pod`, `create_template`, `create_endpoint`, `update_endpoint_template`, and container-registry-auth CRUD before calling them. If a call fails with a field error, re-check the installed SDK source at `site-packages/runpod/api/` — the live SDK is always the source of truth, not this file.

## Safety rules for this account

- Read-only queries: run freely.
- Anything costing money: query `lowestPrice` first, tell the user the $/hr, get explicit approval.
- Always clean up test pods (`terminate_pod`) and report elapsed uptime/cost.
- Remind the user to rotate any API key that was shared in plaintext chat.

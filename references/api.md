# RunPod SDK Full API Reference (runpod-python 1.12.0)

Source of truth: installed SDK at `site-packages/runpod/api/`. All signatures below were copied from that source; read-only functions were additionally verified against the live API.

## Transport layer (`runpod/api/graphql.py`)

```python
runpod.api.graphql.run_graphql_query(query: str, api_key: Optional[str] = None) -> Dict[str, Any]
```
- POSTs to `{RUNPOD_API_BASE_URL or "https://api.runpod.io"}/graphql`
- Headers: `Authorization: Bearer <key>`
- Raises `runpod.error.AuthenticationError` on 401 or missing key; `runpod.error.QueryError` on GraphQL `errors`
- Env overrides: `RUNPOD_API_BASE_URL` (GraphQL host), `RUNPOD_ENDPOINT_BASE_URL` (default `https://api.runpod.ai/v2`)

## User / credentials

```python
runpod.get_user(api_key=None)
# -> {"id": "user_...", "pubKey": "ssh-ed25519 ...", "networkVolumes": [{"id","name","size","dataCenterId"}]}
runpod.update_user_settings(pubkey: str, api_key=None)   # -> {"updateUserSettings": ...}
```

## GPU queries

```python
runpod.get_gpus(api_key=None)
# basic fields per type: id, displayName, memoryInGb   (returns all 48 types)
runpod.get_gpu(gpu_id: str, gpu_quantity: int = 1, api_key=None)
# full fields; raises ValueError if id not found
```

Full field set for `gpuTypes(input: {id: "..."})`:
`maxGpuCount, id, displayName, manufacturer, memoryInGb, cudaCores, secureCloud, communityCloud, securePrice, communityPrice, oneMonthPrice, threeMonthPrice, oneWeekPrice, communitySpotPrice, secureSpotPrice, lowestPrice(input: {gpuCount: N}) { minimumBidPrice uninterruptablePrice }`

## Pods

```python
runpod.get_pods(api_key=None) -> list
runpod.get_pod(pod_id, api_key=None)
```

Pod dict fields (from QUERY_POD): `id, containerDiskInGb, costPerHr, desiredStatus, dockerArgs, dockerId, env, gpuCount, imageName, lastStatusChange, machineId, memoryInGb, name, podType, port, ports, uptimeSeconds, vcpuCount, volumeInGb, volumeMountPath, runtime { ports { ip isIpPublic privatePort publicPort type } }, machine { gpuDisplayName }`

```python
runpod.create_pod(
    name: str,
    image_name: str = "",            # required unless template_id given
    gpu_type_id: str = None,         # None => CPU pod (deployCpuPod)
    cloud_type: str = "ALL",         # ALL | COMMUNITY | SECURE
    support_public_ip: bool = True,
    start_ssh: bool = True,
    data_center_id: str = None,
    country_code: str = None,
    gpu_count: int = 1,
    volume_in_gb: int = 0,
    container_disk_in_gb: int = None,   # defaults to 10 if no template_id
    min_vcpu_count: int = 1,
    min_memory_in_gb: int = 1,
    docker_args: str = "",
    ports: str = None,                  # e.g. "8888/http,666/tcp"
    volume_mount_path: str = "/runpod-volume",
    env: dict = None,                   # {"KEY": "value"}
    template_id: str = None,
    network_volume_id: str = None,      # auto-resolves data_center_id from user's networkVolumes
    allowed_cuda_versions: list = None, # e.g. ["12.4", "12.5"]
    min_download = None,                # Mbps
    min_upload = None,                  # Mbps
    instance_id: str = None,            # CPU pods only, e.g. "cpu3c-2-4"
) -> dict   # {"id", "imageName", "env", "machineId", "machine": {"podHostId"}}

runpod.stop_pod(pod_id)                       # -> {"id", "desiredStatus"}
runpod.resume_pod(pod_id, gpu_count: int)     # gpu_count REQUIRED
runpod.terminate_pod(pod_id)                  # returns None on success
```

Deployment mutation name: `podFindAndDeployOnDemand` (GPU) / `deployCpuPod` (CPU, injects default `templateId: "runpod-ubuntu"`).

## Deployment playbook (11 real pods of lessons, 2026 deployments)

```python
runpod.create_pod(..., start_ssh=True, ports="22/tcp")  # BOTH required for SSH
```

- `create_pod` transient error `There are no longer any instances available...` → retry same call with 20 s backoff (usually lands within 1–3 tries); switching `cloud_type` to `ALL` widens inventory.
- After creation, poll `get_pod(pod_id)["runtime"]["ports"]` — your port appears only once the container is fully up. `uptimeSeconds` may stay 0 forever; ignore it.
- SSH connect: `ssh -i key -o IdentitiesOnly=yes -p <publicPort> root@<ip>` where ip/publicPort come from the `privatePort == 22` entry.
- `isIpPublic: false` (NAT host) → no direct TCP. SSH via `root@<pod_id>.ssh.proxy.runpod.net`; HTTP via `https://<pod_id>-<publicPort>.proxy.runpod.net` (proxy's own 404 = backend not up).
- Key rejected >10 min after ports appear (image/env verified correct) → host-specific startup failure → terminate + recreate (host lottery), or self-heal via docker_args rewriting `/root/.ssh/authorized_keys` from `$PUBLIC_KEY` + `(service ssh start || /usr/sbin/sshd)`.
- `stop_pod` then `resume_pod(pod_id, gpu_count=N)` fails with `There are not enough free GPUs on the host machine` if the GPU was taken — resume is same-host only. Recovery = `terminate_pod` + recreate (container disk lost).
- Image tag re-pushes are silent and can break SSH platform-wide (real incident: all `runpod/pytorch` tags re-pushed 2026-10-30 20:07 UTC, multi-line `PUBLIC_KEY` rejected on new builds). Check `hub.docker.com/v2/repositories/runpod/<img>/tags/<tag>` `last_updated` when auth breaks on every host; pin an older tag.
- After `update_user_settings` (pubkey change), wait a few minutes before `create_pod` — pods created immediately can inject a stale `PUBLIC_KEY`.
- `docker_args` / `env` values pass through the mutation unescaped: escape `"` as `\"` yourself, no stray backslashes. Shell-form execution (multi-command strings work) is confirmed for `runpod/*` images; official `pytorch/pytorch` images exec-form (inline multi-command scripts crash-loop silently — detect via ports never mapping).
- Zero-billing assertion: `runpod.get_pods() == []` (terminated, incl. stopped ones), `get_endpoints() == []`, `get_user()["networkVolumes"] == []`.

## Templates

```python
runpod.create_template(
    name: str,
    image_name: str,
    docker_start_cmd: str = None,
    container_disk_in_gb: int = 10,
    volume_in_gb: int = None,
    volume_mount_path: str = None,
    ports: str = None,
    env: dict = None,
    is_serverless: bool = False,
    registry_auth_id: str = None,
)   # -> saveTemplate {...}
```

## Serverless endpoints

```python
runpod.get_endpoints()
# endpoint fields: aiKey, gpuIds, id, idleTimeout, name, networkVolumeId, locations,
# scalerType, scalerValue, templateId, type, userId, version, workersMax, workersMin,
# workersStandby, gpuCount, env [{key,value}], createdAt, networkVolume {id, dataCenterId}

runpod.create_endpoint(
    name: str,
    template_id: str,
    gpu_ids: str = "AMPERE_16",      # comma-separated GPU family ids
    network_volume_id: str = None,
    locations: str = None,
    idle_timeout: int = 5,
    scaler_type: str = "QUEUE_DELAY",
    scaler_value: int = 4,
    workers_min: int = 0,
    workers_max: int = 3,
    flashboot: bool = False,
    allowed_cuda_versions: str = None,   # comma-separated string
    gpu_count: int = 1,
)   # -> saveEndpoint {...}

runpod.update_endpoint_template(endpoint_id: str, template_id: str)
```

Endpoint client (sync): `runpod.Endpoint(endpoint_id, api_key=None)` with `.run(input: dict) -> Job` and `Job.status() / Job.output(timeout=0) / Job.stream() / Job.cancel(timeout=3)`.
Endpoint client (async): `runpod.AsyncioEndpoint(...)` with `await .run() / .status() / .health() / .purge_queue()`.

## Container registry auth

```python
runpod.create_container_registry_auth(name, username, password)     # -> saveRegistryAuth
runpod.update_container_registry_auth(registry_auth_id, username, password)
runpod.delete_container_registry_auth(registry_auth_id)
```

## REST v2 (verified working)

```python
GET https://api.runpod.io/v2/pods        # Bearer auth -> {"pods": [...], "pagination": {...}}
```
Note: `/v2/account/balance` and `/v2/account` do NOT exist (404). The v6 REST host (`rest.runpod.io/v6/...`) currently redirects to the docs site.

## Error classes (`runpod/error.py`, single file)

```python
class RunPodError(Exception):      # base; .message attribute
class AuthenticationError(RunPodError):   # 401 / missing api_key
class QueryError(RunPodError):            # GraphQL errors; .message + .query attributes
```

## Credential management (`runpod/__init__.py` exports from `cli/groups/config/functions.py`)

```python
runpod.set_credentials(api_key: str, profile: str = "default", overwrite=False) -> None
# writes ~/.runpod/config.toml:  [default]
#                                api_key = "..."
# existing profile raises unless overwrite=True
runpod.check_credentials(profile: str = "default") -> (bool, message)
runpod.get_credentials(profile: str = "default") -> {"api_key": ...} | None
```

## Serverless worker module map (runpod/serverless/, verified in 1.12.0)

| Import path | Public surface |
|---|---|
| `runpod.serverless` | `start(config: dict)` |
| `runpod.serverless.modules.rp_logger` | `RunPodLogger` singleton: `set_level`, `log(message, message_level="INFO", job_id=None)`, `secret(secret_name, secret)`, `debug/info/warn/error/tip/trace(message, request_id=None)` |
| `runpod.serverless.modules.rp_progress` | `progress_update(job: dict, progress) -> None` (async internally) |
| `runpod.serverless.modules.rp_local` | `run_local(config) -> None` (async; runs handler against `--test_input`) |
| `runpod.serverless.modules.rp_fastapi` | `WorkerAPI(config)`, `start_uvicorn(api_host, api_port, api_concurrency)` (used when `--rp_serve_api` or `RUNPOD_REALTIME_PORT` set) |
| `runpod.serverless.utils.rp_download` | `download_files_from_urls(job_id, urls: str|list) -> list[str]`, `file(file_url) -> dict`, `calculate_chunk_size(file_size)` |
| `runpod.serverless.utils.rp_upload` | `bucket_upload(job_id, file_list, bucket_creds)` (boto3), `files(job_id, file_list)`, `upload_file_to_bucket`, `upload_in_memory_object`, `get_boto_client(endpoint_url, ...)` |
| `runpod.serverless.utils.rp_cuda` | `is_available() -> bool` |
| `runpod.serverless.utils.rp_validator` | `validate(raw_input: dict, schema: dict) -> {"validated_input": {...}} or {"errors": [...]}`. Schema item: `{"type": str/int/float/bool, "required": bool, "default": val}`; unknown keys flagged as errors. Verified live: defaults injected correctly. |
| `runpod.serverless.utils.rp_ssrf` | `safe_get(url, ...) -> Response`, `SSRFError(ValueError)`, `resolve_and_validate(host, port)`, `is_safe_address(ip)`, `iter_content_capped(response, chunk_size, max_bytes)` — SSRF-protected fetching |
| `runpod.serverless.utils.rp_cleanup` | `clean(folder_list: list = None)` |
| `runpod.serverless.utils.rp_model_cache` | `resolve_model_cache_path_from_hugginface_repository(...)` |
| `runpod.serverless.utils.rp_debugger` | `Checkpoints`, `LineTimer`, `FunctionTimer`, `get_debugger_output()`, `clear_debugger_output()` |
| `runpod.serverless.utils.rp_volume_cache` | `VolumeCache` class (namespace defaults to `RUNPOD_ENDPOINT_ID`) |

**Does NOT exist in 1.12.0:** the top-level `runpod.validation` module (frequent hallucination from old tutorials) — the real validator is `runpod.serverless.utils.rp_validator.validate`. And the `RUNPOD_API_KEY` env var is never read by the SDK.

## CLI commands (runpod CLI, reads ~/.runpod/config.toml — subcommands verified)

- `runpod config --api_key KEY --profile NAME --check` (wizard)
- `runpod pod list` / `create` / `connect <pod_id>` / `sync <source_pod_id> <dest_pod_id>`
- `runpod exec python <pod_id> <file.py>` (remote python over SSH)
- `runpod project new <name>` / `start` / `deploy`
- `runpod ssh list-keys` / `add-key [--key_file FILE]`

## Serverless job REST URLs (client side, host `https://api.runpod.ai/v2`)

```
POST /<endpoint_id>/run              {"input": {...}}   -> {"id", "status"}
POST /<endpoint_id>/run_sync         (blocks until done; may 504 on long jobs)
GET  /<endpoint_id>/status/<job_id>  -> {"status", "output"}
GET  /<endpoint_id>/health           -> {"jobsInQueue", "workersIdle", ...}
POST /<endpoint_id>/cancel/<job_id>
POST /<endpoint_id>/purge-queue
```
Job statuses: `IN_QUEUE`, `IN_PROGRESS`, `COMPLETED`, `FAILED`, `TIMED_OUT`, `CANCELLED` (final states per `runpod.endpoint.helpers`: COMPLETED/FAILED/TIMED_OUT/CANCELLED).

Sync `Endpoint` client details: `requests.Session` with `Retry(total=5, backoff_factor=1, status_forcelist=[408, 429])`; raises `RuntimeError` on 401; `runpod.Endpoint(endpoint_id, api_key=None)` falls back to `runpod.api_key`.

## Module-level variables (`runpod/__init__.py`)

`runpod.__version__`, `runpod.api_key`, `runpod.profile` (default "default"), `runpod.SSH_KEY_PATH` (default `~/.runpod/ssh`), `runpod.endpoint_url_base` (default `https://api.runpod.ai/v2`).

## Raw GraphQL skeletons (verified)

```graphql
query { gpuTypes { id displayName memoryInGb } }
query { gpuTypes(input: {id: "NVIDIA GeForce RTX 4090"}) { ...full field set above... } }
query pod { pod(input: {podId: "..."}) { ...pod field set above... } }
query myPods { myself { pods { ...pod field set above... } } }
mutation { podStop(input: {podId: "..."}) { id desiredStatus } }
mutation { podResume(input: {podId: "...", gpuCount: 1}) { id desiredStatus } }
mutation { podTerminate(input: {podId: "..."}) }
```

Server error behavior: unknown field names return `{"errors":[{"message":"Cannot query field ...","extensions":{"code":"GRAPHQL_VALIDATION_FAILED"}}]}` and HTTP 200; introspection queries (`__schema`, `__type`) are rejected with `INTROSPECTION_DISABLED`.

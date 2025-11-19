# Local LLM Home Stack

Self-hosted home lab for private LLM chat/search with a Glance dashboard and a secured Flask control API. The stack ships production defaults, GPU-aware docker-compose overlays, and CI that enforces lint + tests from `pyproject.toml` only.

---

## What’s Inside

| Component                | Path                         | Notes |
| ------------------------ | ---------------------------- | ----- |
| Docker stack             | `docker-compose.yml` + optional `docker-compose.gpu.yml` | Ollama, Open WebUI, Glance, SearxNG; GPU overlay auto-applied when NVIDIA runtime is detected. |
| Dashboard config         | `glance/config`, `glance/assets`                         | Glance layouts, widgets, CSS. |
| Control API              | `glance/custom_api_extension`                            | Flask app with token auth, rate limits, rotating logs, platform-aware shutdown/restart. |
| Automation script        | `manage_stack.py`                                         | Creates venv, installs deps from `project.toml`, manages Compose, launches Flask API. |
| Tests                    | `tests/`                                                  | 36 pytest cases covering endpoints, auth, platform detection, subprocess handling. |
| Docs                     | `docs/`                                                   | `gpu-optimization.md`, `windows-autostart.md` (moved from repo root). |
| CI workflows             | `.github/workflows/python-app.yml`, `.github/workflows/pylint.yml` | Ruff + pytest matrix; separate Pylint gate. |

---

## Documentation

- GPU tuning and model tables: `docs/gpu-optimization.md`
- Windows autostart (Task Scheduler + PowerShell): `docs/windows-autostart.md`

---

## Prerequisites

- Python ≥ 3.11 (for tests/CLI) with `pip`
- Docker 24+ with Compose plugin
- NVIDIA Container Toolkit (if using GPU)
- `.env` with at least:
  - `MY_SECRET_TOKEN` (shared with Glance widgets)
  - `MY_PASSWORD` (Werkzeug hash for Glance login)

---

## Fast Start (recommended)

```bash
git clone https://github.com/your-user/local_llm.git
cd local_llm
python manage_stack.py
```

What `manage_stack.py` does:
- Creates/uses `.venv` (configurable via `[tool.manage_stack]` in `project.toml`).
- Installs core + optional dependency groups (`optional_dependency_groups = ["dev"]`).
- Ensures `.env` exists.
- Picks GPU overrides automatically when `docker info` reports an NVIDIA runtime.
- Restarts Docker Compose and launches the Flask API in the foreground for visible logs.

Useful flags:
- `--skip-deps` to skip pip install on restarts
- `--restart-only` to avoid `docker compose down`
- `--clean-shutdown` to stop everything and exit

---

## Manual Bring-up

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -e '.[dev]'
docker compose up -d               # CPU
# or GPU
docker compose -f docker-compose.yml -f docker-compose.gpu.yml up -d
```

Service endpoints:
- Ollama: `http://localhost:11434`
- Open WebUI: `http://localhost:8081`
- Glance: `http://localhost:8090`
- SearxNG: `http://localhost:8082`
- Flask control API: `http://localhost:5001`

---

## Flask Control API (glance/custom_api_extension)

- Token auth: `Authorization: Bearer <token>`, `Authorization: <token>`, form `token`, or `?token=` query.
- Rate limits: default `20/min` global plus `5/min` per endpoint.
- Logging: rotating file `host_flask.log` with ANSI-stripping formatter.
- Platform-aware commands:
  - Shutdown: Linux/WSL/macOS → `sudo shutdown -h now`; Windows → `shutdown /s /f /t 0`
  - Restart: Linux/WSL/macOS → `shutdown -r now`; Windows → `shutdown /r /f /t 0`

Endpoints:
| Route | Method | Description |
| --- | --- | --- |
| `/` | GET | Health check (rate limited) |
| `/shutdown` | POST | Authenticated host shutdown |
| `/restart` | POST | Authenticated host restart |

---

## CI & Quality Gates

- `.github/workflows/python-app.yml`: Ruff lint + pytest on Python 3.11 & 3.12 with coverage artifacts. Dependencies installed from `pyproject.toml` via `pip install -e '.[dev]'` (no `requirements.txt`).
- `.github/workflows/pylint.yml`: Dedicated Pylint job on `glance/` and `manage_stack.py`; add `fail-under` in `pyproject.toml` to enforce a minimum score (e.g., `[tool.pylint.main] fail-under = 9.5`).
- Local commands: `ruff check .`, `pytest`, `pylint glance manage_stack.py`.

---

## Windows Autostart

See `docs/windows-autostart.md` for scheduling `start_services.ps1`. It starts Docker Compose, installs Python deps from `pyproject.toml`, runs the Flask API, and logs to `flask_service.log`.

---

## GPU Optimization

See `docs/gpu-optimization.md` for RTX 5080 defaults, VRAM tables, throughput/context presets, and troubleshooting. The GPU compose overlay is auto-applied when an NVIDIA runtime is detected.

---

## Development Notes

- Code style: `black`, `ruff`
- Lint: `pylint` (scoped to shipped code; extend targets if you want tests linted too)
- Tests: `pytest` (36 passing)
- Packaging: PEP 621 metadata in `project.toml`/`pyproject.toml`; dev extras include `ruff`, `black`, `pylint`, `pytest-cov`, `pip-tools`, `build`.

---

## Changelog (high level)

- CI split into Ruff+pytest and dedicated Pylint workflows; both install from `pyproject.toml`.
- Flask API hardened: clearer logging, rate limits, platform-specific commands, narrowed exception handling.
- `manage_stack.py`: explicit `subprocess.run(check=False)`, docstrings, venv management, GPU auto-detect, configurable flags.
- Docs consolidated under `docs/`; README refreshed.

---

## License

MIT — see `LICENSE`.

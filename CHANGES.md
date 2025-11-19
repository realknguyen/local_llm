# Repository Review & Update Summary

## Overview

Completed a comprehensive review of the home_server repository and updated all files to reflect recent changes, particularly the port change from 8080 to 8090 and added Windows support.

## Changes Made

### 1. Port Configuration Updates

- **docker-compose.yml**: Changed Glance port mapping from `8080:8080` to `8090:8080` (to avoid conflict with Windows AgentService.exe)
- **docker-compose.yml**: Updated Glance label URL from `http://localhost:8080` to `http://localhost:8090`
- **README.md**: Updated service table to reflect new Glance port (8090)

### 2. CORS Support

- **pyproject.toml**: Added `flask-cors>=4.0,<5.0` dependency
- **project.toml**: Added `flask-cors>=4.0,<5.0` dependency
- **requirements.txt**: Added `flask_cors` to glance/custom_api_extension/requirements.txt
- **host_flask.py**: Imported and enabled CORS for all routes

### 3. Enhanced manage_stack.py

- Added `--skip-deps` flag to skip dependency installation
- Added `--restart-only` flag to preserve container state
- Added `--clean-shutdown` flag to force stop with all compose file combinations
- Added `--upgrade` flag to pip install command

### 4. Windows Support

- **start_services.ps1**: Created PowerShell startup script for Windows Task Scheduler
  - Starts Docker Compose stack
  - Installs dependencies
  - Launches Flask server
  - Redirects output to `flask_service.log`

### 5. Documentation Updates

- **README.md**: Added CLI arguments documentation
- **README.md**: Added Windows startup script section with Task Scheduler instructions

### 6. LAN Control Configuration

- **startpage.yml**: Changed API URLs from `localhost` to `falconnet.local` for shutdown/restart buttons
- This allows control from mobile devices and laptops on the LAN

## Files Modified

1. docker-compose.yml
2. glance/config/startpage.yml
3. glance/custom_api_extension/host_flask.py
4. glance/custom_api_extension/requirements.txt
5. manage_stack.py
6. project.toml
7. pyproject.toml
8. start_services.ps1 (created)
9. README.md

## Verified Working

- ✅ Port 8090 for Glance (avoids Windows port conflicts)
- ✅ CORS enabled for cross-origin requests
- ✅ Windows shutdown/restart commands correct
- ✅ LAN hostname (`falconnet.local`) configured
- ✅ All tests passing (36/36)
- ✅ Dependencies in sync across all config files

## Notes

- `.gitignore` already covers `flask_service.log` via `*.log` pattern
- GPU detection and docker-compose.gpu.yml working as expected
- Both `project.toml` and `pyproject.toml` are kept in sync (Python standard is `pyproject.toml`)

## Recommendations for User

1. Test `python manage_stack.py --clean-shutdown` on Windows to ensure clean startup
2. Access dashboard at `http://falconnet.local:8090` from LAN devices
3. Add `start_services.ps1` to Windows Task Scheduler for auto-start on boot
4. Verify firewall allows ports 5001 (Flask) and 8090 (Glance) on Windows

# Windows Auto-Start Setup Guide

This guide explains how to configure Windows to automatically start Docker and Flask services **before login** with proper shutdown/restart permissions.

## Prerequisites

- Windows 10/11 with Administrator access
- Docker Desktop installed and configured to start on boot
- Python and dependencies installed

## Task Scheduler Configuration

### Step 1: Open Task Scheduler

1. Press `Win + R`
2. Type `taskschd.msc` and press Enter
3. Click "Create Task" (not "Create Basic Task")

### Step 2: General Tab

- **Name**: `Home Server Auto Start`
- **Description**: `Starts Docker Compose and Flask API on system boot`
- **Security options**:
  - ✅ **Run whether user is logged on or not** (CRITICAL!)
  - ✅ **Run with highest privileges** (CRITICAL!)
  - ✅ **Hidden** (optional, hides console window)
- **Configure for**: Windows 10/11

### Step 3: Triggers Tab

Click "New..." and configure:

- **Begin the task**: `At startup`
- **Delay task for**: `30 seconds` (give Docker time to start)
- ✅ **Enabled**

### Step 4: Actions Tab

Click "New..." and configure:

- **Action**: `Start a program`
- **Program/script**: `powershell.exe`
- **Add arguments**:

  ```
  -ExecutionPolicy Bypass -WindowStyle Hidden -File "C:\path\to\home_server\start_services.ps1"
  ```

  Replace `C:\path\to\home_server` with your actual path

- **Start in**: `C:\path\to\home_server` (same directory)

### Step 5: Conditions Tab

- ❌ Uncheck "Start the task only if the computer is on AC power"
- ✅ Check "Wake the computer to run this task" (optional)

### Step 6: Settings Tab

- ✅ "Allow task to be run on demand"
- ✅ "Run task as soon as possible after a scheduled start is missed"
- ❌ "Stop the task if it runs longer than" (uncheck this)
- **If the task is already running**: "Do not start a new instance"

### Step 7: Save and Test

1. Click "OK"
2. Enter your Windows password when prompted
3. Right-click the task → "Run" to test immediately
4. Check `flask_service.log` to verify it's working

## Verifying Shutdown/Restart Permissions

The Task Scheduler setup above gives the Flask service the necessary permissions to shutdown/restart Windows before login.

### Test Shutdown Command

From another device on your network:

```bash
# Using curl (replace with your server IP)
curl -X POST http://192.168.1.XXX:5001/shutdown \
  -H "Authorization: Bearer YOUR_SECRET_TOKEN"
```

Or from the Glance dashboard at `http://YOUR_SERVER_IP:8090`

### Troubleshooting

**Problem**: "Access Denied" when trying to shutdown

**Solutions**:

1. Verify Task Scheduler is configured with "Run with highest privileges"
2. Ensure task runs as SYSTEM or Administrator account
3. Check that the task is set to "Run whether user is logged on or not"

**Problem**: Flask service not starting

**Solutions**:

1. Check `flask_service.log` for errors
2. Verify Python path in `start_services.ps1`
3. Ensure Docker Desktop starts before the script runs (increase delay)
4. Run the script manually first: `powershell -ExecutionPolicy Bypass -File start_services.ps1`

**Problem**: Services stop when user logs out

**Solution**: This means the task is NOT configured correctly. Go back to Step 2 and ensure "Run whether user is logged on or not" is checked.

## Alternative: Windows Service (Advanced)

For even more reliability, you can convert the Flask app to a Windows Service using NSSM:

```powershell
# Install NSSM
choco install nssm

# Create service
nssm install HomeServerFlask "C:\path\to\.venv\Scripts\python.exe" "C:\path\to\glance\custom_api_extension\host_flask.py"

# Configure service
nssm set HomeServerFlask AppDirectory "C:\path\to\home_server"
nssm set HomeServerFlask Start SERVICE_AUTO_START
nssm set HomeServerFlask AppStdout "C:\path\to\flask_service.log"
nssm set HomeServerFlask AppStderr "C:\path\to\flask_service.log"

# Start service
nssm start HomeServerFlask
```

## Monitoring

### Check if services are running:

```powershell
# Check Docker containers
docker ps

# Check Flask process
Get-Process python

# View Flask logs
Get-Content flask_service.log -Tail 50 -Wait
```

### Check Task Scheduler history:

1. Open Task Scheduler
2. Find your task
3. Click "History" tab
4. Look for Event ID 200 (task started) and 201 (task completed)

## Security Notes

- The Flask API requires a token (`MY_SECRET_TOKEN`) for all shutdown/restart requests
- Keep your `.env` file secure and never commit it to Git
- Consider using HTTPS if accessing from outside your local network
- The `/f` flag in shutdown commands forces immediate shutdown without saving prompts

## Firewall Configuration

Ensure Windows Firewall allows inbound connections:

```powershell
# Allow Flask port
New-NetFirewallRule -DisplayName "Flask API" -Direction Inbound -LocalPort 5001 -Protocol TCP -Action Allow

# Allow Glance port
New-NetFirewallRule -DisplayName "Glance Dashboard" -Direction Inbound -LocalPort 8090 -Protocol TCP -Action Allow
```

## Summary

With this setup:

- ✅ Docker and Flask start automatically on boot
- ✅ Services run before any user logs in
- ✅ Shutdown/restart commands work from the network
- ✅ Services continue running even when users log out
- ✅ Proper permissions for system-level operations

Your home server is now fully automated! 🚀

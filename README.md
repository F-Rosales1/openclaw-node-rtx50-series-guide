
# Guide: Setting Up a Windows Compute Node for OpenClaw (RTX 5-series)

## 1. Overview & Architecture

This guide details the end-to-end process for configuring a powerful Windows desktop with an NVIDIA RTX GPU to act as a "Compute Node" for an OpenClaw agent running in a separate Linux VM. This setup enables the agent to offload intensive tasks like local LLM inference (Ollama) and image/video generation (ComfyUI) to dedicated hardware.

The connection is established via direct API calls over the local network. This guide was created after determining that the native `openclaw node` and direct `ssh` methods were unreliable in this specific environment, making direct API communication the most robust solution.

## 2. Prerequisites

*   **Host Machine:** A machine running VirtualBox.
*   **VM:** A Linux VM with OpenClaw installed and running.
*   **Compute Node (Desktop):** A Windows 10/11 machine with a modern NVIDIA GPU (e.g., RTX 5-series).
*   **Software (on Desktop):**
    *   Python 3.10+ (ensure it's added to PATH during installation).
    *   Node.js LTS (ensure it's added to PATH during installation).
    *   Git for Windows.

---

## 3. Step-by-Step Configuration

### Step 3.1: VM Network Configuration

*   **Goal:** Place the VM on the same local network as the Windows desktop to allow direct communication.
*   **Action:**
    1.  In VirtualBox, shut down the VM.
    2.  Go to VM Settings > Network.
    3.  Set "Attached to" to **`Bridged Adapter`** (`Adaptador puente`).
    4.  Expand "Advanced" and set "Promiscuous mode" to **`Allow All`** (`Permitir todo`).
    5.  Save and restart the VM.

### Step 3.2: OpenClaw Gateway Configuration (in VM)

*   **Goal:** Allow the OpenClaw Gateway to accept connections from other machines on the network.
*   **Action (in VM terminal):**
    1.  Open the configuration file: `nano ~/.openclaw/openclaw.json`
    2.  Find the `gateway` block and change `"bind": "loopback"` to `"bind": "lan"`.
    3.  Save the file (`Ctrl+S`, `Ctrl+X`).
    4.  Restart the Gateway: `openclaw gateway restart`

### Step 3.3: ComfyUI Installation & GPU Fix (on Windows Desktop)

*   **Goal:** Install ComfyUI and the correct version of PyTorch for your RTX GPU.
*   **Action (in PowerShell):**
    1.  **Create a clean project directory:**
        ```powershell
        cd C:\Users\YourUsername\Documents
        mkdir comfyui_project
        cd comfyui_project
        ```
    2.  **Create a standard Python virtual environment:**
        ```powershell
        py -m venv .venv
        ```
    3.  **Activate the virtual environment:**
        ```powershell
        .\.venv\Scripts\Activate.ps1
        ```
    4.  **Install `uv` and `comfy-cli`:**
        ```powershell
        pip install uv
        uv pip install comfy-cli
        ```
    5.  **Install ComfyUI:**
        ```powershell
        comfy install --nvidia
        ```
    6.  **Fix PyTorch for RTX 5-series (CUDA 13.0+):**
        ```powershell
        pip uninstall torch torchvision torchaudio -y
        uv pip install torch torchvision torchaudio --torch-backend=cu130
        ```

### Step 3.4: Ollama Configuration (on Windows Desktop)

*   **Goal:** Allow the Ollama server to accept connections from other machines on the network.
*   **Action (in PowerShell):**
    1.  **Stop the Ollama service** (Right-click the icon in the system tray and "Quit").
    2.  **Set the host environment variable:**
        ```powershell
        [System.Environment]::SetEnvironmentVariable('OLLAMA_HOST', '0.0.0.0', 'User')
        ```
    3.  **Restart the Ollama application** from your Start Menu.

### Step 3.5: Windows Firewall Configuration

*   **Goal:** Open ports for Ollama and ComfyUI.
*   **Action (in PowerShell - Run as Administrator):**
    ```powershell
    New-NetFirewallRule -DisplayName "Ollama (Port 11434)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 11434
    New-NetFirewallRule -DisplayName "ComfyUI (Port 8188)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 8188
    ```

---

## 4. Usage & Automation

### 4.1. Manual Service Startup

To use this setup, the services must be running on your Windows desktop.

*   **To run Ollama:** Ensure the Ollama application is running (launched from your Start Menu).
*   **To run ComfyUI (in PowerShell):**
    ```powershell
    cd C:\Users\YourUsername\Documents\comfyui_project
    .\.venv\Scripts\Activate.ps1
    comfy launch -- --listen 0.0.0.0 --port 8188
    ```
    *Keep this window open while you need to generate images.*

### 4.2. Automation Script (`start_node.ps1`)

To automate startup, create a file named `start_node.ps1` on your desktop with the following content. You can then right-click it and "Run with PowerShell".

```powershell
# ===================================================================
#  OpenClaw Compute Node Startup Script
#
#  This script starts the necessary services on this Windows machine
#  to act as a compute node for the OpenClaw agent.
#
#  - Ensures the Ollama service is running.
#  - Launches the ComfyUI server in a new PowerShell window.
# ===================================================================

Write-Host "--- Starting OpenClaw Compute Node Services ---" -ForegroundColor Green

# 1. Check for Ollama
Write-Host "Checking for Ollama service..."
$ollamaProcess = Get-Process -Name "ollama" -ErrorAction SilentlyContinue

if ($null -eq $ollamaProcess) {
    Write-Host "Ollama service not found. Please start the Ollama application from your Start Menu." -ForegroundColor Yellow
} else {
    Write-Host "Ollama service is running."
}

# 2. Start ComfyUI in a new window
Write-Host "Launching ComfyUI server in a new PowerShell window..." -ForegroundColor Green

# Define the script block to be executed in the new window
$comfyScriptBlock = {
    Write-Host "--- ComfyUI Server ---"
    Write-Host "Activating virtual environment and starting ComfyUI..."
    cd C:\Users\YourUsername\Documents\comfyui_project
    .\.venv\Scripts\Activate.ps1
    comfy launch -- --listen 0.0.0.0 --port 8188
    # This window will remain open. Close it to stop the ComfyUI server.
}

# Use Start-Process to launch a new PowerShell window that runs the script block
Start-Process powershell -ArgumentList "-NoExit", "-Command", "& {$comfyScriptBlock}"

Write-Host ""
Write-Host "--- Node Startup Initiated ---" -ForegroundColor Green
Write-Host "ComfyUI is launching in a new window. Please keep that window open."
Write-Host "You can now close this script window."
Read-Host -Prompt "Press Enter to exit"
```

## 5. Verification

*   **From your VM, you should be able to reach the APIs:**
    ```bash
    curl http://<Your-Desktop-IP>:11434/api/tags  # Should list Ollama models
    curl http://<Your-Desktop-IP>:8188          # Should return ComfyUI's HTML
    ```

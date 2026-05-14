<!-- cSpell:disable -->
<!-- markdownlint-disable -->
# 📖 User Manual: The AI-SOAR Analyst
**Version:** 1.0 (n8n + Local LLM Orchestration)

This document is the official manual for System Administrators and SOC Analysts to configure and operate the full AI-SOAR Analyst system.

---

## 🏗️ 1. System Overview
This system is designed to reduce the workload of SOC Analysts by using AI to automatically read and filter logs:
1. **Wazuh** triggers an alert when a threat is detected.
2. **n8n** receives the alert and forwards the data to **Ollama (Llama 3.2)** for analysis.
3. The AI summarizes the context, assesses the risk, and sends a notification to **Telegram**.
4. The Analyst clicks an interactive response button (e.g., Block IP) directly in the chat.
5. n8n catches the button click and executes an API call to Wazuh to immediately ban the attacker.

---

## 🚀 2. System Startup
1. Open your Terminal / PowerShell and navigate to the project folder `SOC_The AI_SOAR Analyst`.
2. Run the following command to start the n8n and AI servers:
   ```powershell
   docker-compose up -d
   ```
3. **CRITICAL:** You must download the Llama 3.2 AI model to your local machine (only required once):
   ```powershell
   docker exec -it ai_soar_ollama ollama pull llama3.2
   ```

---

## ⚙️ 3. n8n Configuration
1. Open your Web Browser and go to `http://localhost:5678`.
2. Create your Owner Account.
3. Navigate to the **Workflows** menu on the left ➡️ Click **Add Workflow**.
4. Click the **three dots (⋯)** icon in the top right corner ➡️ Select **Import from File**.
5. Choose the file `n8n/workflow_ai_soar.json`.
6. **Configure the 3 Credentials points as follows:**
   * **Point 1:** Double-click the `Get Wazuh Token` Node.
     - Click "Create New Credential" (HTTP Basic Auth).
     - Enter the Username / Password for your **Wazuh Manager**.
     - Click Save.
   * **Point 2:** Double-click the `Telegram (Human-in-the-loop)` Node.
     - Click "Create New Credential".
     - Enter the **Bot Token** from `@BotFather`.
     - Enter your **Chat ID** (can be obtained via `@userinfobot`).
   * **Point 3:** Double-click the `Execute Block IP (Wazuh AR)` Node.
     - Scroll down to the URL field and change `wazuh-manager` to **your Wazuh server's actual IP address** (e.g., `https://192.168.1.100:55000/active-response`).
7. Toggle the switch in the top right corner to **Active**.

---

## 🛡️ 4. Wazuh Manager Integration
For Wazuh to forward alerts to n8n, you must configure your Wazuh Manager server:

1. SSH into your Wazuh Manager server.
2. Edit the configuration file:
   ```bash
   sudo nano /var/ossec/etc/ossec.conf
   ```
3. Copy the snippet below and paste it **before** the final `</ossec_config>` tag:
   ```xml
   <integration>
     <name>custom-n8n</name>
     <!-- Replace [IP_N8N] with the IP of the machine running Docker n8n -->
     <hook_url>http://[IP_N8N]:5678/webhook/wazuh-webhook</hook_url>
     <level>7</level>
     <alert_format>json</alert_format>
   </integration>
   ```
4. Restart the Wazuh service to apply changes:
   ```bash
   sudo systemctl restart wazuh-manager
   ```

---

## 👨‍💻 5. Daily Operations for SOC Analysts
Once configured, the system will run automatically 24/7. Your role is simply to:

1. **Wait for Alerts:** When a high-severity alert (Level 7+) occurs, you will receive an instant Telegram message.
2. **Review AI Summary:** The AI will summarize the event, determine if it is a False Positive, and recommend an action.
3. **Make a Decision (Human-in-the-loop):**
   * If it's a real threat: Click `🚨 Block IP`. The system will automatically block the IP.
   * If it's a false alarm: Click `✅ Ignore`. The system will safely dismiss the alert.
4. **Confirm Action:** Upon successfully blocking an IP, the bot will reply with *"🛡️ The IP has been successfully blocked by Wazuh Active Response."*

---

> [!TIP]
> **Troubleshooting:**
> If the bot does not send a message after an alert, go to n8n ➡️ Double-click the rightmost node ➡️ Select the **Executions** tab. You will be able to clearly see the workflow history and where any errors occurred.

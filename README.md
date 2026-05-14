<!-- cSpell:disable -->
<!-- markdownlint-disable -->
# 🛡️ The AI-SOAR Analyst

**The AI-SOAR Analyst** is an enterprise-grade Security Orchestration, Automation, and Response (SOAR) tool. It leverages Local LLMs (via Ollama) to analyze security alerts from Wazuh in real-time, filter out False Positives, and provides interactive response capabilities through Telegram via **n8n** orchestration.

## 🚀 Features
- **Zero-Code Orchestration**: Powered by n8n for highly visual, maintainable, and flexible workflows.
- **Local AI Analysis**: Uses Llama 3.2 via native Ollama integration for 100% private, offline threat intelligence.
- **Enterprise Telegram UI**: Richly formatted alerts with inline interactive buttons for Analyst decision-making.
- **Interactive SOAR (Human-in-the-loop)**: 
    - 🚨 **Block IP**: Trigger active response to ban malicious IPs directly from Telegram.
    - ✅ **Ignore**: Acknowledge and dismiss False Positives.
- **Automated Triage**: Initial AI-driven filtering out of clear False Positives to reduce alert fatigue.

## 🏗️ Architecture
The system operates on a containerized microservices architecture:
- `docker-compose.yml`: Deploys **n8n** and **Ollama** within a secure isolated internal network.
- `n8n/workflow_ai_soar.json`: The core SOAR automation logic that connects Wazuh, Llama 3.2, and Telegram.

## 🛠️ Installation & Setup

### 1. Prerequisites
- **Docker & Docker Compose** installed.
- **Wazuh**: Manager installed and accessible on your network.
- **Telegram Bot**: Created via `@BotFather`.

### 2. Deployment
```powershell
# Clone the repository
git clone https://github.com/Taeaps561/AI-SOAR-Docker.git
cd AI-SOAR-Docker

# Start the SOAR infrastructure
docker-compose up -d

# Download the Llama 3.2 model into the Ollama container
docker exec -it ai_soar_ollama ollama pull llama3.2
```

### 3. n8n Configuration
1. Open your browser and navigate to `http://localhost:5678`.
2. Complete the initial Owner setup.
3. Go to **Workflows** -> **Add Workflow** -> **Import from File**.
4. Select the `n8n/workflow_ai_soar.json` file.
5. Double-click the **Telegram** nodes to add your Bot Token and Chat ID.
6. Double-click the **Execute Block IP** node to insert your Wazuh API Token.
7. Toggle the workflow to **Active**.

### 4. Wazuh Integration
Add the following block to your Wazuh Manager's `ossec.conf` to forward alerts to n8n:
```xml
<integration>
  <name>custom-n8n</name>
  <hook_url>http://[N8N_IP]:5678/webhook/wazuh-webhook</hook_url>
  <level>7</level>
  <alert_format>json</alert_format>
</integration>
```
Restart the Wazuh Manager to apply changes.

## 📄 License
MIT License. Created for SOC Analysts by The AI-SOAR Team.

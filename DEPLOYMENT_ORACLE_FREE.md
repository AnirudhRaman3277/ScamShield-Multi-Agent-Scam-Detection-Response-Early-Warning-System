# Live Deployment Guide — Oracle Cloud Always Free

## Objective

Run the project continuously from a cloud VM, rather than from `localhost`. The Docker Compose stack starts:

- n8n, with persistent workflow/execution storage
- Ollama, with a locally hosted `llama3.2:3b` model
- an automatic model-download job

No paid LLM or search API key is required for the included Phase 1 workflow. The workflow uses Google News RSS for initial discovery and local Ollama for reasoning.

## Important limits

This is suitable for a college demo, not high-volume production. Oracle Always Free capacity can be unavailable in a selected region, and a 3B model on a 2-OCPU VM will be relatively slow. Create only resources marked **Always Free** and monitor the OCI console so that paid resources are not created accidentally.

## Before deployment

You need:

1. An Oracle Cloud account with an Always Free ARM VM available in its home region.
2. An SSH key pair.
3. A secured public hostname/domain if n8n must be reachable directly by evaluators. Otherwise, use SSH port forwarding for the demo.
4. A Telegram Bot token and channel ID only when Phase 3 Telegram integration is enabled.

## 1. Create the VM

In Oracle Cloud, create an Always Free `VM.Standard.A1.Flex` Ubuntu instance. Allocate no more than the Always Free allowance across all ARM instances. For this demo, use 2 OCPUs and 12 GB RAM if available. Add an ingress rule only for SSH (port 22) initially.

Copy the VM public IP after creation.

## 2. Connect and install Docker

From PowerShell, replace placeholders:

```powershell
ssh -i <path-to-private-key> ubuntu@<server-ip>
```

On the VM:

```bash
sudo apt update
sudo apt install -y git docker.io docker-compose-plugin
sudo usermod -aG docker $USER
exit
```

Reconnect after the group change.

## 3. Clone and configure the repository

```bash
git clone <your-github-repository-url> agentic-ai-flexi
cd agentic-ai-flexi
cp .env.example .env
```

Generate a stable encryption key and add it to `.env`:

```bash
openssl rand -hex 32
nano .env
```

Set `N8N_ENCRYPTION_KEY` to that generated value. Keep the included Ollama settings unless you intentionally select another installed model.

## 4. Start and verify

```bash
docker compose up -d
docker compose ps
docker compose logs -f ollama-pull-model
```

The first model download can take several minutes. Confirm the local model API from the VM:

```bash
docker compose exec ollama ollama list
```

Then access n8n securely over an SSH tunnel from your laptop:

```powershell
ssh -i <path-to-private-key> -L 5678:localhost:5678 ubuntu@<server-ip>
```

Open `http://localhost:5678` in your browser and create the n8n owner account. Import the workflows under `n8n/workflows/`.

## 5. Make it publicly reachable only if required

Do not expose port 5678 directly without HTTPS and authentication. For a public evaluator link, use a domain plus an HTTPS reverse proxy or a managed tunnel, then set n8n's public host/webhook URL accordingly. Keep n8n user management enabled and use a strong owner password.

For a demo where evaluators are physically present, the SSH tunnel above is safer and sufficient to demonstrate the live cloud deployment.

## 6. Keep it running

The `restart: unless-stopped` policy restarts n8n and Ollama after a VM/Docker restart. Before demo day run:

```bash
cd ~/agentic-ai-flexi
docker compose ps
docker compose logs --tail=100 n8n
docker compose exec ollama ollama list
```

If a service is unhealthy, restart it with:

```bash
docker compose restart n8n ollama
```

Do not delete Docker volumes; they retain n8n configuration and downloaded model data.

## Evidence for CA3

Capture the Oracle VM instance page, `docker compose ps`, n8n execution log, agent messages/state updates, the Ollama model listing, and a successful Telegram test-channel post after Phase 3 is connected. These are real deployment/execution artifacts, unlike diagrams or illustrative traces.

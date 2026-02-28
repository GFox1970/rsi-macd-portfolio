# Gemini 2.0 (Google GenAI) Setup Guide

This guide walks you through setting up Gemini 2.0 authentication for the trading bot using the lightweight `google-genai` SDK.

## Prerequisites

- Google Cloud project with billing enabled: `gen-lang-client-0536875321`
- Google API Key (Gemini API) from [Google AI Studio](https://aistudio.google.com/)
- Local terminal access or SSH access to your Hetzner VM

## Step 1: Enable Vertex AI API

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Select your project: `gen-lang-client-0536875321`
3. Navigate to **APIs & Services** → **Library**
4. Search for "Vertex AI API"
5. Click **ENABLE**

## Step 2: Create Service Account

1. In Google Cloud Console, go to **IAM & Admin** → **Service Accounts**
2. Click **CREATE SERVICE ACCOUNT**
3. Fill in the details:
   - **Service account name**: `trading-bot-vertex-ai`
   - **Description**: `Service account for trading bot Gemini API access`
4. Click **CREATE AND CONTINUE**

5. Grant the following role:
   - **Role**: `Vertex AI User`
6. Click **CONTINUE**, then **DONE**

## Step 3: Generate JSON Key

1. Find your newly created service account in the list
2. Click on it to open details
3. Go to the **KEYS** tab
4. Click **ADD KEY** → **Create new key**
5. Select **JSON** format
6. Click **CREATE**

The JSON key file will download automatically (e.g., `trading-bot-vertex-ai-xxxxx.json`).

**⚠️ IMPORTANT**: Keep this file secure! Never commit it to git.

## Step 4: Configure Local Environment

### Option A: Local Development

1. **Move the JSON key file** to a secure location:
   ```bash
   mkdir -p ~/rsi-macd-bot/config
   mv ~/Downloads/trading-bot-vertex-ai-*.json ~/rsi-macd-bot/config/vertex-ai-key.json
   chmod 600 ~/rsi-macd-bot/config/vertex-ai-key.json
   ```

2. **Update `.env` file**:
   ```bash
   cd ~/rsi-macd-bot
   nano .env
   ```

   Add these lines:
   ```bash
   # Vertex AI Configuration
   GOOGLE_CLOUD_PROJECT=gen-lang-client-0536875321
   GOOGLE_APPLICATION_CREDENTIALS=/app/config/vertex-ai-key.json
   VERTEX_AI_LOCATION=global
   
   # Optional: Keep for backward compatibility during testing
   # GOOGLE_API_KEY=your-old-ai-studio-key
   ```

3. **Update `.gitignore`** to prevent accidental commits:
   ```bash
   echo "config/vertex-ai-key.json" >> .gitignore
   echo "config/*.json" >> .gitignore
   ```

4. **Install updated dependencies**:
   ```bash
   pip install -r requirements.txt
   ```

### Option B: Hetzner VM Deployment

1. **Copy the JSON key to your VM**:
   ```bash
   scp ~/Downloads/trading-bot-vertex-ai-*.json user@your-hetzner-vm:/home/user/rsi-macd-bot/config/vertex-ai-key.json
   ```

2. **SSH into your VM** and update the`.env` file:
   ```bash
   ssh user@your-hetzner-vm
   cd ~/rsi-macd-bot
   nano .env
   ```

   Add the same configuration:
   ```bash
   GOOGLE_CLOUD_PROJECT=gen-lang-client-0536875321
   GOOGLE_APPLICATION_CREDENTIALS=/app/config/vertex-ai-key.json
   VERTEX_AI_LOCATION=global
   ```

3. **Set proper permissions**:
   ```bash
   chmod 600 ~/rsi-macd-bot/config/vertex-ai-key.json
   ```

## Step 5: Update Docker Configuration

The `docker-compose.yml` has already been updated to mount the credentials. Verify it includes:

```yaml
services:
  orchestrator:
    volumes:
      - ./config:/app/config:ro  # ← Should be present
    env_file:
      - .env
```

No additional changes needed if this is already in place.

## Step 6: Configure GitHub Actions

For the scheduled orchestrator workflow:

1. **Encode the JSON key as base64**:
   ```bash
   cat config/vertex-ai-key.json | base64 -w 0 > vertex-ai-key-base64.txt
   ```

2. **Add to GitHub Secrets**:
   - Go to your repository → Settings → Secrets and variables → Actions
   - Click **New repository secret**
   - Name: `VERTEX_AI_SERVICE_ACCOUNT_JSON`
   - Value: Paste the base64-encoded content from `vertex-ai-key-base64.txt`
   - Click **Add secret**

3. **Add additional secrets**:
   - Name: `GOOGLE_CLOUD_PROJECT`
   - Value: `gen-lang-client-0536875321`

The workflow file will be updated separately to use these secrets.

## Step 7: Request Quota Increases

Default Vertex AI quotas may still be too low for heavy batch operations.

1. Go to [Google Cloud Console → Quotas](https://console.cloud.google.com/iam-admin/quotas)
2. Filter by:
   - **Service**: Vertex AI API
   - **Location**: europe-west1
3. Look for these quotas:
   - `Generate Content requests per minute per region per base model`
   - `Generate Content input tokens per minute per region per base model`
   - `Generate Content output tokens per minute per region per base model`

4. For each quota:
   - Click the checkbox next to it
   - Click **EDIT QUOTAS** at the top
   - Enter your desired limit (e.g., 300 for RPM)
   - Provide justification: "Automated trading bot requiring real-time market analysis"
   - Submit the request

**Typical approval time**: Instant to a few hours for reasonable increases.

## Step 8: Verify Setup

### Test Locally

```bash
cd ~/rsi-macd-bot
export GOOGLE_CLOUD_PROJECT=gen-lang-client-0536875321
export GOOGLE_APPLICATION_CREDENTIALS=$PWD/config/vertex-ai-key.json
export VERTEX_AI_LOCATION=global

# Test the Vertex AI client
python -c "from trading_bot.core.vertex_ai_client import VertexAIClient; c = VertexAIClient(); print('Available:', c.available)"
```

Expected output:
```
✅ Vertex AI initialized: gemini-2.0-flash (project=gen-lang-client-0536875321, location=global)
Available: True
```

### Test with Sentinel Agent

```bash
# Run a simple sentinel test
python agent/sentinel_agent.py
```

If you see logs like:
```
Sentinel initialized with Vertex AI: gemini-2.0-flash
```

Then the setup is working! ✅

## Troubleshooting

### Error: "Permission denied" on JSON key file

```bash
chmod 600 config/vertex-ai-key.json
```

### Error: "Vertex AI not available"

Check that:
1. `GOOGLE_CLOUD_PROJECT` is set correctly
2. `GOOGLE_APPLICATION_CREDENTIALS` points to the correct path **inside the container** (use `/app/config/vertex-ai-key.json`, not your local path)
3. The JSON key file exists and is mounted properly in Docker

### Error: "Authentication failed"

Verify:
1. The service account has the `Vertex AI User` role
2. The Vertex AI API is enabled in your project
3. The JSON key is valid and not expired

### Still getting 429 errors

You may still need to request quota increases (see Step 7). Default quotas are higher than AI Studio but may not be enough for batch operations.

## Cost Monitoring

Monitor your Vertex AI usage and costs:

1. Go to [Google Cloud Console → Billing → Reports](https://console.cloud.google.com/billing/reports)
2. Filter by **Service**: Vertex AI API
3. View usage by day

**Typical costs** (as of 2026):
- Gemini 2.0 Flash: ~$0.075 per 1M input tokens, ~$0.30 per 1M output tokens
- Expected daily cost for your bot: $0.10 - $2.00 depending on activity

## Comparing AI Studio vs Vertex AI

| Feature | AI Studio (Old) | Vertex AI (New) |
|---------|----------------|-----------------|
| Authentication | API Key | Service Account JSON |
| Default RPM Quota | 15 (hard limit) | 60+ |
| Max RPM (with increase) | 15 (cannot increase) | 300+ |
| Billing | Cannot bill properly | Full billing support |
| Production Ready | No | Yes |
| Cost | Same | Same |

## Next Steps

After setup is complete:
1. Remove or comment out old `GOOGLE_API_KEY` from `.env`
2. Uninstall old package: `pip uninstall google-generativeai`
3. Monitor the first few orchestrator runs for any auth issues
4. Request additional quota increases if needed

---

**Support**: If you encounter issues, check the [Vertex AI documentation](https://cloud.google.com/vertex-ai/docs) or open an issue in the repository.

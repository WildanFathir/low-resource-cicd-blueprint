# 🔒 GitHub Repository Secrets Configuration Guide

This blueprint pipeline utilizes **GitHub Repository Secrets** to securely store sensitive data such as server IP addresses, cryptographic keys, and access tokens. By leveraging Secrets, your confidential credentials are encrypted at rest and will not be exposed within your commit history or public GitHub Actions workflow logs.

Please prepare and configure the following mandatory environment variables before triggering the pipeline:

---

## 📋 Secret Variables Reference & Retrieval Guide

### 1. `SERVER_IP`

- **Description:** The public IPv4 address of your target remote server / VPS.
- **How to Retrieve:** 1. Log in to your cloud provider dashboard (e.g., Biznet GIO, DigitalOcean, AWS, etc.). 2. Navigate to your Instance or Virtual Machine details page. 3. Copy the listed Public IPv4 address (e.g., `103.x.x.x`).

### 2. `SERVER_USER`

- **Description:** The administrative username used to establish an SSH connection to your VPS operating system.
- **How to Retrieve:** \* Use the default username provided by your VPS operating system distribution (e.g., `ubuntu`, `debian`, or `root`).
  - Alternatively, input the custom username if you configured a non-default user during the initial server setup.

### 3. `SERVER_SSH_KEY`

- **Description:** The plain text content of your computer's private SSH key that has been authorized on the VPS.
- **How to Retrieve:** 1. Open a terminal session on your local development machine. 2. Display the private key content by running the following command (ensure you are copying the private identity file, **not** the public key file ending in `.pub`):
  ```bash
  cat ~/.ssh/id_ed25519
  # Or for older RSA deployments: cat ~/.ssh/id_rsa
  ```

  3. Copy the entire terminal text block output, precisely including the header `-----BEGIN OPENSSH PRIVATE KEY-----` and the footer `-----END OPENSSH PRIVATE KEY-----`.

### 4. `GHCR_TOKEN`

- **Description:** A Personal Access Token (PAT) from your GitHub account that grants your remote server the authorization to pull built container images from the GitHub Container Registry (GHCR).
- **How to Retrieve:**
  1. Navigate to your GitHub profile settings (Click your profile picture in the top-right corner $\rightarrow$ select **Settings**).
  2. Scroll down on the left sidebar menu and click **Developer Settings**.
  3. Select **Personal access tokens** $\rightarrow$ click **Tokens (classic)**.
  4. Click the **Generate new token** dropdown $\rightarrow$ select **Generate new token (classic)**.
  5. Provide a descriptive note (e.g., `vps-deployment-token`).
  6. Under the scopes checklist selection, ensure you explicitly check:
     - `write:packages` (this will automatically provision `read:packages`).
  7. Click **Generate token** at the bottom of the page.
  8. **CRITICAL STEP:** Immediately copy the generated token string (prefixed with `ghp_...`). GitHub will only present this string once.

### 5. `DISCORD_WEBHOOK`

- **Description:** The automated Webhook integration URL generated from your Discord server channel to broadcast real-time deployment lifecycle alerts.
- **How to Retrieve:**
  1. Open your Discord application and enter the server you manage.
  2. Right-click on the designated text channel intended for DevOps monitoring logs $\rightarrow$ select **Edit Channel**.
  3. Navigate to the **Integrations** tab on the left sidebar.
  4. Click on **Webhooks** or the **Create Webhook** action button.
  5. Customize your integration profile name (e.g., `DevOps Bot`), and click **Copy Webhook URL**.

### 6. `DOMAIN_NAME`

- **Description:** The fully qualified public domain name or subdomain mapped to resolve to your application endpoint (e.g., `example.com`).
- **How to Retrieve:** * Provide the official domain string of your project, ensuring its DNS *A Record\* configuration points directly to your `SERVER_IP`.

### 7. `ENV_FILE`

- **Description:** The complete environment configurations file data (`.env`) compiled explicitly for your production server environment.
- **How to Retrieve:**
  - Copy the entire text block inside your local project's environment configuration file, ensuring all values (such as production database credentials, production application keys, etc.) are properly populated for the target server deployment.

---

## 🗄️ Storing Secrets in GitHub Repository

Once you have gathered all the configuration secret values above, follow these procedural steps to register them securely within your project dashboard:

1. Navigate to the landing page of your code repository hosted on GitHub.
2. Click on the **Settings** tab located in the repository's top navigation asset bar.
3. On the left sidebar menu layout, locate the **Security** category dropdown $\rightarrow$ click **Secrets and variables** $\rightarrow$ select **Actions**.
4. Ensure the **Secrets** management tab is active, then click the green **New repository secret** button.
5. Populate the encryption form fields:
   - **Name:** Input the precise variable flag name using uppercase alphanumeric characters (e.g., `SERVER_IP`).
   - **Secret:** Paste the corresponding raw configuration value retrieved from the steps above.
6. Click the **Add secret** button to finalize the encryption storage block.
7. Repeat steps 4 through 6 sequentially until all 7 mandatory secret parameters are securely provisioned within your Actions control board.

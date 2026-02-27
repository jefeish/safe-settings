# Running Safe-settings with GitHub Actions (GHA)

This guide describes how to schedule a full safe-settings sync using GitHub Actions. This assumes that an `admin` repository has been configured with your `safe-settings` configuration. Refer to the [How to Use](../README.md#how-to-use) docs for more details on that process.


## GitHub App Creation
Follow the [Create the GitHub App](deploy.md#create-the-github-app) guide to create an App in your GitHub account. This will allow `safe-settings` to access and modify your repos.


## Defining the GitHub Action Workflow
Running a full-sync with `safe-settings` can be done via `npm run full-sync`. This requires installing Node, such as with [actions/setup-node](https://github.com/actions/setup-node) (see example below). When doing so, the appropriate environment variables must be set (see the [Environment variables](#environment-variables) document for more details).


### Example GHA Workflow
The below example uses the GHA "cron" feature to run a full-sync every 4 hours. While not required, this example uses the `.github` repo as the `admin` repo (set via `ADMIN_REPO` env var) and the safe-settings configurations are stored in the `safe-settings/` directory (set via `CONFIG_PATH` and `DEPLOYMENT_CONFIG_FILE`).

```yaml
name: Safe Settings Sync
on:
  schedule:
    - cron: "0 */4 * * *"
  workflow_dispatch: {}

jobs:
  safeSettingsSync:
    runs-on: ubuntu-latest
    env:
      # Version/tag of github/safe-settings repo to use:
      SAFE_SETTINGS_VERSION: 2.1.17

      # Path on GHA runner box where safe-settings code downloaded to:
      SAFE_SETTINGS_CODE_DIR: ${{ github.workspace }}/.safe-settings-code
    steps:
      # Self-checkout of 'admin' repo for access to safe-settings config:
      - uses: actions/checkout@v4

      # Checkout of safe-settings repo for running full sync:
      - uses: actions/checkout@v4
        with:
          repository: github/safe-settings
          ref: ${{ env.SAFE_SETTINGS_VERSION }}
          path: ${{ env.SAFE_SETTINGS_CODE_DIR }}
      - uses: actions/setup-node@v4
      - run: npm install
        working-directory: ${{ env.SAFE_SETTINGS_CODE_DIR }}
      - run: npm run full-sync
        working-directory: ${{ env.SAFE_SETTINGS_CODE_DIR }}
        env:
          GH_ORG: ${{ vars.SAFE_SETTINGS_GH_ORG }}
          APP_ID: ${{ vars.SAFE_SETTINGS_APP_ID }}
          PRIVATE_KEY: ${{ secrets.SAFE_SETTINGS_PRIVATE_KEY }}
          GITHUB_CLIENT_ID: ${{ vars.SAFE_SETTINGS_GITHUB_CLIENT_ID }}
          GITHUB_CLIENT_SECRET: ${{ secrets.SAFE_SETTINGS_GITHUB_CLIENT_SECRET }}
          ADMIN_REPO: .github
          CONFIG_PATH: safe-settings
          DEPLOYMENT_CONFIG_FILE: ${{ github.workspace }}/safe-settings/deployment-settings.yml
```

## Troubleshooting

### "HttpError: Not Found" Error (Even When Files Exist)

If you're getting a 404 Not Found error even though your configuration files exist in the correct location, this is typically a **GitHub App permissions issue**. Here's how to diagnose and fix:

#### 1. Verify GitHub App Installation
- Go to your organization settings → GitHub Apps → Installed GitHub Apps
- Ensure your safe-settings app is installed and has access to the `.github` repository
- If using a personal account, check Settings → Applications → Installed GitHub Apps

#### 2. Check Repository Permissions
Your GitHub App needs these permissions:
- **Repository permissions:**
  - Contents: Read & Write (to read config files and update repos)
  - Metadata: Read (to access repository information)
  - Administration: Write (to modify repository settings)
  - Pull requests: Write (for creating PR comments)
  - Issues: Write (for creating error issues)

#### 3. Verify App Installation Scope
- If the app is installed on "Selected repositories", make sure `.github` repository is included
- Consider installing on "All repositories" for easier management

#### 4. Check Environment Variables
Ensure these match your setup:
```yaml
env:
  GH_ORG: your-organization-name        # Must match exactly
  ADMIN_REPO: .github                   # Repository containing configs  
  CONFIG_PATH: safe-settings           # Directory containing config files
```

#### 5. Debug with Additional Logging
Add this environment variable to enable debug logging:
```yaml
env:
  LOG_LEVEL: debug
```

### "Cannot read properties of undefined (reading 'data')" Error  
This error was caused by a bug in error handling and has been fixed. Update your safe-settings code to the latest version.

### Webhook Warning: "repository_ruleset is not a known webhook name"
This is a harmless warning that can be ignored.

## Quick Verification Steps

Before running the workflow, verify your setup:

1. **Test GitHub App Access:**
   ```bash
   # Using GitHub CLI (if available)
   gh api repos/OWNER/.github/contents/safe-settings/settings.yml
   ```

2. **Verify Files Exist:**
   - Navigate to `https://github.com/YOUR_ORG/.github/tree/main/safe-settings`
   - Confirm `settings.yml` and `deployment-settings.yml` are present

3. **Check App Permissions:**
   - Go to `https://github.com/organizations/YOUR_ORG/settings/installations` 
   - Click on your safe-settings app
   - Verify it has access to the `.github` repository
   - Check that required permissions are granted

4. **Test Locally (Optional):**
   ```bash
   # Clone and test locally
   git clone https://github.com/github/safe-settings
   cd safe-settings
   npm install
   # Set environment variables
   export GH_ORG="your-org"
   export ADMIN_REPO=".github" 
   export CONFIG_PATH="safe-settings"
   # ... other env vars
   npm run full-sync
   ```

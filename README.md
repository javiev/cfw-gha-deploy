# Cloudflare Workers Deployment via GitHub Actions Issue

This repository demonstrates and solves a common deployment issue when using **GitHub Actions** with `wrangler.json` configuration files for Cloudflare Workers.

> **Note**: This issue **only occurs when deploying via GitHub Actions**. If you connect your repository directly to Cloudflare Workers through the Cloudflare Dashboard, deployments work without any issues. This problem is specific to the `cloudflare/wrangler-action@v3` used in CI/CD pipelines.

> **Package Manager**: While this repository focuses on **pnpm**, there is evidence that the same issue occurs with **npm** and other package managers. The root cause is the outdated Wrangler version in the action, not the package manager itself.

## The Problem

When deploying Cloudflare Workers using `cloudflare/wrangler-action@v3` with pnpm and a `wrangler.json` configuration file, the deployment fails with one of the following errors:

### Error 1: Missing pnpm executable

If you don't install pnpm before the wrangler-action:

```
Run cloudflare/wrangler-action@v3
🔍 Checking for existing Wrangler installation
📥 Installing Wrangler
Error: Unable to locate executable file: pnpm. Please verify either the file path exists or the file can be found within a directory specified by the PATH environment variable. Also check the file mode to verify the file is executable.
Error: 🚨 Action failed
```

### Error 2: Missing entry-point

If pnpm is installed but Wrangler 3.90.0 (default) cannot read `wrangler.json`:

```
Run cloudflare/wrangler-action@v3
🔍 Checking for existing Wrangler installation
📥 Installing Wrangler
  ✅ Wrangler installed
🚀 Running Wrangler Commands
   ⛅️ wrangler 3.90.0 (update available 4.55.0)
  ---------------------------------------------
✘ [ERROR] Missing entry-point: The entry-point should be specified via the command line 
(e.g. `wrangler deploy path/to/script`) or the `main` config field.

Error: The process '/home/runner/setup-pnpm/node_modules/.bin/pnpm' failed with exit code 1
Error: 🚨 Action failed
```

### Root Cause

The `cloudflare/wrangler-action@v3` currently installs **Wrangler 3.90.0 by default**, which has the following limitations:

- **Wrangler 3.90.0 and earlier**: Only support `wrangler.toml` configuration files
- **Wrangler 3.91.0+**: Added support for both `wrangler.json`/`wrangler.jsonc` and `wrangler.toml` configuration files
- **Wrangler 4.x**: Continues full support for both `.toml` and `.json` formats

Since modern Cloudflare Workers projects (especially those created with libraries like Hono) use `wrangler.json` by default, the action fails to find the entry point because it cannot read the JSON configuration format.

### Related Issues

This issue has been reported multiple times in the wrangler-action repository:
- [Issue #390](https://github.com/cloudflare/wrangler-action/issues/390) - Cannot use wranglerVersion to use latest v4 release
- [Issue #379](https://github.com/cloudflare/wrangler-action/issues/379) - Support for wrangler.json configuration
- [Pull Request #363](https://github.com/cloudflare/wrangler-action/pull/363) - Attempt to add Wrangler 4.x support


## The Solution

The solution is to **manually install Wrangler 4.x before the wrangler-action runs**, ensuring the action uses a version that supports `wrangler.json`.

### Working GitHub Actions Workflow

```yaml
name: Deploy Worker

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 60
    steps:
      - uses: actions/checkout@v4
      
      - name: Install pnpm
        uses: pnpm/action-setup@v4
        with:
          version: '10'
      
      - name: Install Wrangler latest v4
        run: pnpm add -D wrangler@latest
      
      - name: Build & Deploy Worker
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
```

### How It Works

1. **Checkout code**: Downloads your repository code to the GitHub Actions runner
2. **Install pnpm**: Sets up pnpm (version 10.x) since GitHub runners don't include it by default
3. **Install Wrangler 4**: Manually installs the latest Wrangler 4.x which supports `wrangler.json`
4. **Deploy**: The wrangler-action detects the existing Wrangler installation and uses it for deployment

> **Note for npm users**: If you're using npm instead of pnpm, replace the pnpm installation step with npm commands. The core issue (outdated Wrangler version) and solution (install Wrangler 4.x) remain the same regardless of package manager.

### Important Notes

- **This configuration solves the deployment error** by ensuring a compatible Wrangler version is used
- **The workflow can be adjusted** according to your project's specific requirements and optimizations
- You may want to add additional steps like:
    - Dependency caching for faster builds
    - Running tests before deployment
    - Installing all project dependencies if needed
    - Environment-specific deployments

## Project Structure

```
.
├── src/
│   └── index.ts          # Worker entry point
├── wrangler.json         # Wrangler configuration (JSON format)
├── package.json          # Project dependencies
├── pnpm-lock.yaml        # pnpm lockfile
└── .github/
    └── workflows/
        └── deploy.yml    # GitHub Actions workflow
```

## Setup Instructions

1. **Clone this repository**
   ```bash
   git clone git@github.com:javiev/cfw-gha-deploy.git
   cd cfw-gha-deploy
   ```

2. **Install dependencies**
   ```bash
   pnpm install
   ```

3. **Configure GitHub Secrets**

   Add the following secrets to your GitHub repository (Settings → Secrets and variables → Actions):
    - `CLOUDFLARE_API_TOKEN`: Your Cloudflare API token with Workers deploy permissions
    - `CLOUDFLARE_ACCOUNT_ID`: Your Cloudflare account ID

   To create an API token:
    - Go to [Cloudflare Dashboard](https://dash.cloudflare.com/profile/api-tokens)
    - Click "Create Token"
    - Use the "Edit Cloudflare Workers" template
    - Select your account and zones

4. **Push to main branch**
   ```bash
   git push origin main
   ```

   The GitHub Action will automatically deploy your Worker to Cloudflare.

## Why This Happens

The `cloudflare/wrangler-action@v3` was designed when `wrangler.toml` was the standard configuration format. The action's default Wrangler version (3.90.0) predates the introduction of `wrangler.json` support in Wrangler 3.91.0+.


### Why Cloudflare Dashboard Deployments Work

When you connect your GitHub repository to Cloudflare Workers through the Cloudflare Dashboard (Git integration), Cloudflare's infrastructure uses the **latest version of Wrangler** automatically. This means:

- ✅ Cloudflare Dashboard deployments use Wrangler 4.x by default
- ✅ Full support for `wrangler.json` configuration files
- ✅ No manual configuration needed

**The issue only affects manual GitHub Actions workflows** because the `wrangler-action@v3` is locked to an older Wrangler version by default.

## Alternative: Convert to wrangler.toml

If you prefer not to modify your GitHub Actions workflow, you can convert your `wrangler.json` to `wrangler.toml`:

**wrangler.json:**
```json
{
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2025-12-10"
}
```

**wrangler.toml:**
```toml
name = "my-worker"
main = "src/index.ts"
compatibility_date = "2025-12-10"
```

However, this approach goes against the modern convention used by current Cloudflare Workers templates and frameworks.

## Test Branches

This repository includes test branches that demonstrate the problem and solution:

### Branch: `add-gha` - Initial Setup (Fails)
**[View branch](https://github.com/javiev/cfw-gha-deploy/tree/add-gha)**

This branch follows the standard `wrangler-action` documentation without any modifications. It demonstrates the deployment failure when using `wrangler.json` with the default action configuration.

**Result**: ❌ Deployment fails with the entry-point error

### Branch: `cfw-gha-fix` - Working Solution
**[View branch](https://github.com/javiev/cfw-gha-deploy/tree/cfw-gha-fix)**

This branch implements the fix by manually installing Wrangler 4.x before running the wrangler-action. It includes multiple test commits showing the solution in action.

**Result**: ✅ Deployment succeeds

### View Workflow Runs
**[See all GitHub Actions runs](https://github.com/javiev/cfw-gha-deploy/actions)**

You can view the actual workflow executions to see the failures and successes in action. Compare the workflows between these branches to see exactly what changes are needed to fix the issue.

## Resources

- [Wrangler Configuration Documentation](https://developers.cloudflare.com/workers/wrangler/configuration/)
- [wrangler-action GitHub Repository](https://github.com/cloudflare/wrangler-action)
- [Issue #390: Cannot use latest v4 release](https://github.com/cloudflare/wrangler-action/issues/390)
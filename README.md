# Sidecar Loader Buildpack

A Cloud Native Buildpack that downloads and sets up sidecar containers from private GitHub repositories.

## Features

- Downloads and installs GitHub CLI during build
- Downloads and installs Node.js runtime for sidecar applications
- Clones sidecar code from private GitHub repositories
- Supports authentication via GitHub tokens
- Configurable repository, branch, and path selection via build-time environment variables
- Automatic Node.js dependency installation (npm install/npm ci)
- Intelligent Node.js application startup detection
- Automatic detection of sidecar startup scripts

## Usage

### 1. Build-Time Configuration

Configure the sidecar repository using build-time environment variables:

```bash
# Required: GitHub repository containing the sidecar code
export BP_SIDECAR_REPO="your-org/your-private-sidecar-repo"

# Optional: Branch to clone (defaults to "main")
export BP_SIDECAR_BRANCH="main"

# Optional: Path within the repository (defaults to ".")
export BP_SIDECAR_PATH="sidecar"
```

### 2. GitHub Authentication

The buildpack supports multiple ways to provide GitHub authentication:

#### Option 1: Environment Variable
```bash
export GITHUB_TOKEN_ENV="ghp_your_token_here"
```

#### Option 2: Platform Environment (CNB)
Set `GITHUB_TOKEN` in your platform environment.

#### Option 3: Platform Bindings (CNB)
Provide the token via platform bindings at `bindings/github/token`.

### 3. Application Configuration (Optional)

You can still include `sidecar.yml` or `sidecar.yaml` files in your application for runtime configuration:

```yaml
# This file will be copied to the sidecar layer for runtime use
sidecar:
  name: "my-sidecar"
  environment:
    SIDECAR_MODE: "production"
    LOG_LEVEL: "info"
  ports:
    - 8080
    - 9090
```

### 4. Sidecar Code Structure

Your sidecar repository can contain:

**For Node.js Applications:**
- `package.json` - Node.js application manifest (required)
- `package-lock.json` - Dependency lock file (recommended)
- `index.js`, `server.js`, or `app.js` - Main application file
- `start` script in package.json (recommended)
- Any other Node.js application files

**For Shell Script Applications:**
- `start.sh` or `run.sh` - Main startup script (will be executed automatically)
- Any other files needed by your sidecar

The buildpack will:
1. Install Node.js runtime (v20.10.0)
2. Clone your repository
3. Copy the specified path to the sidecar layer
4. Install Node.js dependencies (if package.json exists)
5. Make shell scripts executable
6. Execute the appropriate startup method during container launch

### 5. Build Process

The buildpack will:

1. Install GitHub CLI
2. Install Node.js runtime (v20.10.0)
3. Authenticate with GitHub using the provided token
4. Clone the repository specified by `BP_SIDECAR_REPO`
5. Copy sidecar files to the appropriate layer
6. Install Node.js dependencies (if package.json exists)
7. Set up environment variables and PATH
8. Create a sidecar loader script for runtime

### 6. Runtime

At runtime, the sidecar loader will:
1. Check for sidecar configuration
2. Set up Node.js environment and PATH
3. Look for and execute startup method in this order:
   - `start.sh` or `run.sh` (shell scripts)
   - `npm start` (if start script exists in package.json)
   - Main file from package.json
   - `index.js`, `server.js`, or `app.js` (fallbacks)
4. Set up the sidecar environment

## Environment Variables

### Build-Time Configuration
- `BP_SIDECAR_REPO`: GitHub repository (required)
- `BP_SIDECAR_BRANCH`: Branch to clone (optional, defaults to "main")
- `BP_SIDECAR_PATH`: Path within repository (optional, defaults to ".")

### Authentication
- `GITHUB_TOKEN_ENV`: GitHub personal access token
- `GITHUB_TOKEN`: Alternative token variable name
- Platform bindings: `bindings/github/token`

### Runtime Variables (Set by buildpack)
- `SIDECAR_LOADER_PATH`: Path to the sidecar layer
- `SIDECAR_REPO`: The repository that was cloned
- `SIDECAR_BRANCH`: The branch that was used
- `NODE_VERSION`: Version of Node.js installed
- `PATH`: Updated to include sidecar, Node.js, and GitHub CLI paths

## Example Usage

### With pack CLI:
```bash
# Set configuration
export BP_SIDECAR_REPO="mycompany/monitoring-sidecar"
export BP_SIDECAR_BRANCH="v1.2.3"
export BP_SIDECAR_PATH="dist"
export GITHUB_TOKEN_ENV="ghp_xxxxxxxxxxxxxxxxxxxx"

# Build
pack build my-app --buildpack path/to/sidecar-loader
```

### With project.toml:
```toml
[[build.env]]
name = "BP_SIDECAR_REPO"
value = "mycompany/monitoring-sidecar"

[[build.env]]
name = "BP_SIDECAR_BRANCH"
value = "v1.2.3"

[[build.env]]
name = "BP_SIDECAR_PATH"
value = "dist"

[[build.env]]
name = "GITHUB_TOKEN_ENV"
value = "ghp_xxxxxxxxxxxxxxxxxxxx"
```

### With Dockerfile (multi-stage build):
```dockerfile
FROM paketobuildpacks/builder:base as build
ENV BP_SIDECAR_REPO=mycompany/monitoring-sidecar
ENV BP_SIDECAR_BRANCH=v1.2.3
ENV GITHUB_TOKEN_ENV=ghp_xxxxxxxxxxxxxxxxxxxx
COPY . /workspace
RUN /cnb/lifecycle/creator -app=/workspace my-app
```

## Detection

The buildpack will be applied if any of the following conditions are met:
1. `BP_SIDECAR_REPO` environment variable is set
2. `sidecar.yml` or `sidecar.yaml` file exists in the app directory
3. `.sidecar` marker file exists
4. `ENABLE_SIDECAR` environment variable is set

## Node.js Sidecar Example

Here's an example Node.js sidecar structure:

**package.json:**
```json
{
  "name": "my-sidecar",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "express": "^4.18.0"
  }
}
```

**index.js:**
```javascript
const express = require('express');
const app = express();
const port = process.env.PORT || 8080;

app.get('/health', (req, res) => {
  res.json({ status: 'healthy', timestamp: new Date().toISOString() });
});

app.listen(port, () => {
  console.log(`Sidecar listening on port ${port}`);
});
```

The buildpack will automatically:
1. Install dependencies with `npm ci` (or `npm install`)
2. Start the application with `npm start`

## Troubleshooting

- Ensure your GitHub token has access to the private repository
- Check that the repository, branch, and path exist
- For Node.js sidecars, ensure `package.json` is present and valid
- Verify that your sidecar has a `start.sh`, `run.sh`, or proper Node.js entry point
- Review build logs for authentication, cloning, or dependency installation errors
- Use `BP_SIDECAR_REPO` environment variable for repository configuration
- Check Node.js version compatibility (buildpack installs v20.10.0)

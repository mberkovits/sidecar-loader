# How to Use the Sidecar Loader Buildpack

## Quick Test

1. Create a test application:
```bash
mkdir test-app && cd test-app
echo 'console.log("Hello World");' > app.js
echo '{"name":"test","main":"app.js","scripts":{"start":"node app.js"}}' > package.json
echo "# Trigger sidecar buildpack" > .sidecar
```

2. Build with the sidecar buildpack:
```bash
pack build test-app --buildpack /path/to/sidecar-loader --builder paketobuildpacks/builder:base
```

## With Actual Sidecar

1. Set environment variables:
```bash
export BP_SIDECAR_REPO="your-org/your-private-sidecar-repo"
export GITHUB_TOKEN_ENV="ghp_your_github_token"
```

2. Build your application:
```bash
pack build my-app --buildpack /path/to/sidecar-loader --builder paketobuildpacks/builder:base
```

## Common Issues

- **"Image not found"**: You're trying to build the buildpack itself. Build an application instead.
- **"Buildpack not applied"**: Ensure you have a sidecar trigger (BP_SIDECAR_REPO, .sidecar file, etc.)
- **"Authentication failed"**: Check your GITHUB_TOKEN_ENV is valid and has repo access.

## Directory Structure

```
your-project/
├── app.js              # Your main application
├── package.json        # Your app dependencies
├── .sidecar           # Triggers the buildpack (or use BP_SIDECAR_REPO)
└── (other app files)
```

NOT:
```
sidecar-loader/         # This is the buildpack, don't build this
├── bin/
├── buildpack.toml
└── ...
```


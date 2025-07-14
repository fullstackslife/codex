# Build and Deploy Instructions

This document provides comprehensive instructions for building, testing, and deploying the OpenAI Codex CLI.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Environment Setup](#environment-setup)
- [Development Workflow](#development-workflow)
- [Building the Project](#building-the-project)
- [Testing](#testing)
- [Docker Deployment](#docker-deployment)
- [Production Release](#production-release)
- [CI/CD Pipeline](#cicd-pipeline)
- [Troubleshooting](#troubleshooting)

## Prerequisites

### System Requirements

- **Node.js**: Version 22 or newer (LTS recommended)
- **Operating Systems**: macOS 12+, Ubuntu 20.04+/Debian 10+, or Windows 11 via WSL2
- **Package Manager**: pnpm 10.8.1 or higher
- **Git**: Version 2.23+ (recommended for built-in PR helpers)
- **RAM**: 4GB minimum (8GB recommended)
- **Docker**: Required for containerized deployment

### Tools Installation

```bash
# Install Node.js 22+ (using Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install 22
nvm use 22

# Install pnpm globally
npm install -g pnpm@10.8.1

# Or enable corepack (available with Node.js 22+)
corepack enable
corepack prepare pnpm@10.8.1 --activate

# Verify installations
node --version    # Should be 22.x.x
pnpm --version    # Should be 10.8.1
```

## Environment Setup

### 1. Clone the Repository

```bash
git clone https://github.com/openai/codex.git
cd codex
```

### 2. Install Dependencies

```bash
# Install all workspace dependencies
pnpm install

# This will install dependencies for:
# - Root workspace (development tools)
# - codex-cli package (main CLI application)
```

### 3. Environment Configuration

Create a `.env` file in the project root for local development:

```bash
# .env
OPENAI_API_KEY=your-api-key-here
DEBUG=false
```

## Development Workflow

### Code Quality Tools

The project uses several tools to maintain code quality:

- **ESLint**: Code linting and style enforcement
- **Prettier**: Code formatting
- **TypeScript**: Type checking
- **Vitest**: Unit testing
- **Husky**: Git hooks for pre-commit and pre-push checks

### Git Hooks

Husky is configured with the following hooks:

- **Pre-commit**: Runs lint-staged to format and lint files before committing
- **Pre-push**: Runs tests and type checking before pushing to remote

### Development Commands

```bash
# Format code
pnpm run format:fix

# Lint code
pnpm run lint:fix

# Type check
pnpm run typecheck

# Run tests in watch mode
pnpm --filter @openai/codex run test:watch

# Development build with watch mode
pnpm --filter @openai/codex run dev

# Development build and run
pnpm --filter @openai/codex run build:dev
```

## Building the Project

### Standard Build

```bash
# Build the entire project
pnpm run build

# This runs: pnpm --filter @openai/codex run build
# Which executes: node build.mjs in codex-cli directory
```

### Build Process Details

The build process uses **esbuild** via `build.mjs` and:

1. Compiles TypeScript source files from `src/` directory
2. Bundles the application into a single JavaScript file
3. Generates source maps for debugging
4. Handles special cases (like react-devtools-core import stripping)
5. Outputs to `codex-cli/dist/cli.js`

### Build Outputs

After building, you'll find:

```
codex-cli/
├── dist/
│   ├── cli.js          # Main compiled bundle
│   └── cli.js.map      # Source map for debugging
├── bin/
│   └── codex.js        # Entry point script
```

### Development vs Production Builds

**Development Build** (`--dev` flag or `NODE_ENV=development`):

- No minification
- Inline source maps for better stack traces
- Shebang enables Node's source-map support

**Production Build** (default):

- Minified output
- External source maps
- Optimized for distribution

## Testing

### Running Tests

```bash
# Run all tests
pnpm run test

# Run tests in watch mode (for development)
pnpm --filter @openai/codex run test:watch

# Run specific test file
pnpm --filter @openai/codex run test tests/specific-test.test.ts
```

### Test Suite Coverage

The project includes comprehensive tests for:

- Text buffer operations
- Agent functionality (thinking time, termination, network errors, cancellation)
- Multiline input handling
- Patch application
- Project documentation parsing
- Rate limiting and error handling
- Terminal UI components

### Test Framework

- **Vitest**: Fast unit test runner
- **Ink Testing Library**: For testing terminal UI components
- **React Testing**: For React component testing

## Docker Deployment

### Building Docker Image

```bash
# Navigate to codex-cli directory
cd codex-cli

# Build the container (includes npm pack and docker build)
./scripts/build_container.sh
```

### Docker Build Process

The `build_container.sh` script:

1. Installs dependencies (`npm install`)
2. Builds the project (`npm run build`)
3. Creates npm package (`npm pack`)
4. Builds Docker image with the package

### Running in Container

```bash
# Run codex in a sandboxed container
./scripts/run_in_container.sh "your command here"

# Run with specific work directory
./scripts/run_in_container.sh --work_dir /path/to/project "command"
```

### Container Features

- **Network Sandboxing**: Firewall rules restrict network access to OpenAI API only
- **Filesystem Isolation**: Limited write access to working directory
- **Security**: Runs as non-root user with minimal privileges

### Firewall Configuration

The container uses `init_firewall.sh` to:

1. Set up iptables rules for network isolation
2. Allow DNS resolution and localhost communication
3. Permit access only to `api.openai.com`
4. Block all other outbound network traffic
5. Verify firewall rules are working correctly

## Production Release

### Release Process

The release process is automated through npm scripts:

```bash
# Full release (from codex-cli directory)
pnpm run release

# Or step by step:
pnpm run release:readme     # Copy README.md
pnpm run release:version    # Bump version with timestamp
pnpm install                # Update lockfile
pnpm run release:build-and-publish  # Build and publish to npm
```

### Version Management

Versions follow the pattern: `0.1.{timestamp}` where timestamp is `YYMMDDHHMI`.

### Publishing to npm

The package is published as `@openai/codex` to npm registry:

```bash
# Manual publish (after build)
npm publish

# The package includes:
# - README.md
# - bin/ (entry scripts)
# - dist/ (compiled code)
# - src/ (source code)
```

### Installation Methods

**From npm (recommended):**

```bash
npm install -g @openai/codex
```

**From source:**

```bash
git clone https://github.com/openai/codex.git
cd codex/codex-cli
corepack enable
pnpm install
pnpm build
pnpm link  # Link globally
```

## CI/CD Pipeline

### GitHub Actions Workflow

The CI pipeline (`.github/workflows/ci.yml`) runs on:

- Pull requests to `main` branch
- Pushes to `main` branch

### CI Steps

1. **Environment Setup**

   - Ubuntu latest runner
   - Node.js 22
   - pnpm 10.8.1
   - Dependency caching

2. **Quality Checks**

   - Code formatting (`pnpm run format`)
   - Linting with strict rules
   - TypeScript type checking
   - Unit tests

3. **Build Verification**
   - Production build
   - Build artifact validation

### CI Configuration

```yaml
# Key CI settings
timeout-minutes: 10
node-version: 22
pnpm-version: 10.8.1
max-warnings: -1 # Strict linting
```

## Troubleshooting

### Common Issues

**Node.js Version Mismatch:**

```bash
# Error: Unsupported engine: wanted: {"node":">=22"}
# Solution: Upgrade to Node.js 22+
nvm install 22
nvm use 22
```

**pnpm Installation Issues:**

```bash
# If pnpm install fails:
rm -rf node_modules pnpm-lock.yaml
pnpm install
```

**Build Failures:**

```bash
# Clean build
rm -rf codex-cli/dist
pnpm run build
```

**Test Failures:**

```bash
# Run tests with verbose output
pnpm --filter @openai/codex run test -- --reporter=verbose
```

**Docker Issues:**

```bash
# Rebuild container
docker rmi codex
cd codex-cli
./scripts/build_container.sh
```

### Development Environment

**Nix Flake Development (Optional):**

```bash
# Enter Nix development shell
nix develop

# Build with Nix
nix build

# Run via Nix flake
nix run .#codex
```

### Memory Issues

If you encounter memory issues during build or test:

```bash
# Increase Node.js memory limit
export NODE_OPTIONS="--max-old-space-size=4096"

# Or set in CI environment
NODE_OPTIONS: --max-old-space-size=4096
```

### Git Hooks Issues

If Husky hooks fail:

```bash
# Reinstall hooks
pnpm run prepare

# Skip hooks temporarily
git commit --no-verify
```

---

## Additional Resources

- [Main README](./README.md) - Usage and configuration
- [PNPM Migration Guide](./PNPM.md) - Package manager details
- [Contributing Guidelines](./README.md#contributing) - Development guidelines
- [Husky Documentation](./codex-cli/HUSKY.md) - Git hooks setup
- [Examples](./codex-cli/examples/) - Usage examples and templates

For issues and support, please refer to the [GitHub Issues](https://github.com/openai/codex/issues) page.

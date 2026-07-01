# PyTorch POWER Backend CI/CD

This repository contains the GitHub Actions workflow for building and testing PyTorch on IBM POWER architecture (ppc64le).

## Architecture Overview

The CI/CD pipeline uses a hybrid approach to work around GitHub Actions' lack of native ppc64le runner support:

```
GitHub Event
      │
      ▼
GitHub-hosted ubuntu-latest
      │
      ├── Checkout PyTorch
      ├── Archive source
      ├── Copy source to Power LPAR (scp/rsync)
      ├── ssh ailiblpar1
      │      ├── Extract source
      │      ├── Create venv
      │      ├── Build PyTorch
      │      └── Run tests
      └── Copy artifacts back
```

## Workflow Details

The workflow (`power-crcr-ci.yml`) performs the following steps:

1. **GitHub-hosted Runner (ubuntu-latest)**:
   - Checks out the PyTorch source code
   - Archives the source into a tarball
   - Transfers the archive to the Power LPAR via SCP

2. **Power LPAR (via SSH)**:
   - Extracts the source code
   - Creates a Python virtual environment
   - Installs dependencies
   - Builds PyTorch wheel
   - Runs smoke tests

3. **GitHub-hosted Runner (ubuntu-latest)**:
   - Copies build artifacts back from Power LPAR
   - Uploads artifacts to GitHub Actions
   - Cleans up remote workspace

## Required GitHub Secrets

To use this workflow, you must configure the following secrets in your GitHub repository:

### `POWER_SSH_USER`
- **Description**: SSH username for connecting to the Power LPAR
- **Example**: `pytorch-ci`
- **Setup**: Go to Settings → Secrets and variables → Actions → New repository secret

### `POWER_SSH_PRIVATE_KEY`
- **Description**: SSH private key for authentication to the Power LPAR
- **Format**: Full private key including headers
- **Example**:
  ```
  -----BEGIN OPENSSH PRIVATE KEY-----
  b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
  ...
  -----END OPENSSH PRIVATE KEY-----
  ```
- **Setup**: 
  1. Generate an SSH key pair on your local machine: `ssh-keygen -t rsa -b 4096 -C "pytorch-ci"`
  2. Copy the public key to the Power LPAR: `ssh-copy-id user@ailiblpar1.pperf.tadn.ibm.com`
  3. Copy the entire private key content and add it as a GitHub secret

## Environment Variables

The workflow uses the following environment variables:

- `POWER_HOST`: Hostname of the Power LPAR (default: `ailiblpar1.pperf.tadn.ibm.com`)
- `REMOTE_WORK_DIR`: Temporary directory on the Power LPAR for build operations
- `PYTORCH_REPO_URL`: URL of the PyTorch repository

## Prerequisites on Power LPAR

The Power LPAR must have the following installed:

- Python 3.8 or later
- `python3-venv` package
- Build tools (gcc, g++, make)
- Git (for submodules)
- Sufficient disk space for PyTorch builds (~10GB recommended)

## Triggering the Workflow

The workflow is triggered by:

- **Pull Requests**: Opened, synchronized, or reopened on `main`, `master`, or `release/**` branches
- **Push Events**: Commits pushed to `main`, `master`, or `release/**` branches
- **Repository Dispatch**: External webhook events with types `pull_request` or `push`

## Artifacts

Build artifacts (PyTorch wheel files) are uploaded to GitHub Actions and can be downloaded from the workflow run page.

## Troubleshooting

### SSH Connection Issues
- Verify the `POWER_SSH_USER` and `POWER_SSH_PRIVATE_KEY` secrets are correctly configured
- Ensure the Power LPAR is accessible from GitHub's IP ranges
- Check that the SSH key has been added to `~/.ssh/authorized_keys` on the Power LPAR

### Build Failures
- Check the Power LPAR has sufficient resources (CPU, memory, disk space)
- Verify all build dependencies are installed
- Review the build logs in the GitHub Actions workflow run

### Cleanup Issues
- The workflow automatically cleans up the remote workspace after each run
- If manual cleanup is needed, SSH to the Power LPAR and remove `/tmp/pytorch-ci-*` directories

## Security Considerations

- SSH private keys are stored as GitHub encrypted secrets
- The workflow uses `StrictHostKeyChecking=no` for automated connections (consider using known_hosts for production)
- Remote workspace directories are unique per workflow run to prevent conflicts
- Cleanup steps run even if the workflow fails to prevent disk space issues

## Contributing

When modifying the workflow:

1. Test changes in a fork first
2. Ensure all secrets are properly referenced
3. Verify cleanup steps execute correctly
4. Update this README if new secrets or prerequisites are added
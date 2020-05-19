# AWS Practice Project for Node

## Purpose
1. Provides a template for building Node Applications for AWS using Visual Studio Code.
2. A containerized development environment using the [Remote-Containers](https://github.com/Microsoft/vscode-remote-release.git) extension.

## Requirements
1. Visual Studio Code
2. Remote - Containers extension for Visual Studio Code
3. Docker

## Installed Frameworks
1. AWS CLI
2. AWS SAM CLI
3. AWS Toolkit for VS Code
4. Docker (Docker In Docker)
5. Git 
6. Ruby
7. Curl, Wget

## Expectation
1. Development in run using a non-root user account (without sudo capabilities) - i.e. `vscode` user.
2. The container application inherits host's AWS Configuration. The  `~/.aws` folder is accessibled from the container.
3. The container app reuses ssh private key configuration from the host.  SSH-AGENT settings are forwarded to the container application.

## Variants of containerization
Method is configured via `VSC_DIND` environment variables, in `devcontainer.json` file.
| Value     | Description                                                                                         |
|-----------|-----------------------------------------------------------------------------------------------------|
| `root`    | Docker CLI in container accesses host Docker Engine (root access required)                          |
| `nonroot` | Docker CLI in container accesses host Docker Eingine (nonroot access method)                        |
| `dind`    | Start Docker Engine within container when the container starts up. CLI uses container Docker Engine |
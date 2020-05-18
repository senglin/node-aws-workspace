# AWS Practice Project for Node

## Purpose
1. Provides a template for building Node Applications for AWS using Visual Studio Code.
2. A containerized development environment using the [Remote-Containers](https://github.com/Microsoft/vscode-remote-release.git) extension.

## Installed Frameworks
1. AWS CLI
2. AWS SAM CLI
3. AWS Toolkit for VS Code
4. Docker (Docker In Docker)
5. Git 
6. Ruby
7. Curl, Wget

## Expectation
1. The container application inherits host's AWS Configuration. The  `~/.aws` folder is accessibled from the container.
2. The container app reuses ssh private key configuration from the host.  SSH-AGENT settings are forwarded to the container application.
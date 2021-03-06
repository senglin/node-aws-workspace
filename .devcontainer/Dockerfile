#-------------------------------------------------------------------------------------------------------------
# Copyright (c) Microsoft Corporation. All rights reserved.
# Licensed under the MIT License. See https://go.microsoft.com/fwlink/?linkid=2090316 for license information.
#-------------------------------------------------------------------------------------------------------------

FROM golang:1 AS fargatedist

# Avoid warnings by switching to noninteractive
ENV DEBIAN_FRONTEND=noninteractive

# Configure apt, install packages and tools
RUN apt-get update \
    && apt-get -y install --no-install-recommends apt-utils dialog 2>&1 \
    #
    # Verify git, process tools, lsb-release (common in install instructions for CLIs) installed
    && apt-get -y install git openssh-client zip vim less iproute2 procps lsb-release upx file make \
    #
    # SSL dependencies
    && apt-get -y install apt-transport-https ca-certificates gnupg2 \
    #
    # Clean up
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/* /tmp/gotools

WORKDIR /fargate

RUN git clone -b combined-fixes https://github.com/senglin/fargatecli.git .
RUN go mod download
RUN GOOS=linux GOARCH=amd64 go build -o fargate main.go

# Switch back to dialog for any ad-hoc use of apt-get
ENV DEBIAN_FRONTEND=dialog

#-------------------------------------------------------------------------------------------------------------

FROM node:14

# The node image includes a non-root user with sudo access. Use the "remoteUser"
# property in devcontainer.json to use it. On Linux, the container user's GID/UIDs
# will be updated to match your local UID/GID (when using the dockerFile property).
# See https://aka.ms/vscode-remote/containers/non-root-user for details.
ARG USERNAME=vscode
ARG USER_UID=1001
ARG USER_GID=$USER_UID
ARG NONROOT_USER=$USERNAME

ARG NPM_GLOBAL=/usr/local/share/npm-global
ENV PATH=${PATH}:${NPM_GLOBAL}/bin
ENV NVM_DIR=/usr/local/share/nvm

# Options for common package install script
ARG INSTALL_ZSH="true"
ARG UPGRADE_PACKAGES="true"
ARG COMMON_SCRIPT_SOURCE="https://raw.githubusercontent.com/microsoft/vscode-dev-containers/master/script-library/common-debian.sh"
ARG COMMON_SCRIPT_SHA="dev-mode"
ENV VISUAL=vim
ENV EDITOR=vim

# Configure apt and install packages
RUN apt-get update \
    && export DEBIAN_FRONTEND=noninteractive \
    #
    # Verify git, common tools / libs installed, add/modify non-root user, optionally install zsh
    && apt-get -y install --no-install-recommends curl vim ca-certificates 2>&1 \
    && curl -sSL  ${COMMON_SCRIPT_SOURCE} -o /tmp/common-setup.sh \
    && ([ "${COMMON_SCRIPT_SHA}" = "dev-mode" ] || (echo "${COMMON_SCRIPT_SHA} */tmp/common-setup.sh" | sha256sum -c -)) \
    && /bin/bash /tmp/common-setup.sh "${INSTALL_ZSH}" "${USERNAME}" "${USER_UID}" "${USER_GID}" "${UPGRADE_PACKAGES}" \
    && rm /tmp/common-setup.sh \
    #
    # Remove outdated yarn from /opt and install via package 
    # so it can be easily updated via apt-get upgrade yarn
    && rm -rf /opt/yarn-* \
    && rm -f /usr/local/bin/yarn \
    && rm -f /usr/local/bin/yarnpkg \
    && apt-get install -y curl apt-transport-https lsb-release \
    && curl -sS https://dl.yarnpkg.com/$(lsb_release -is | tr '[:upper:]' '[:lower:]')/pubkey.gpg | apt-key add - 2>/dev/null \
    && echo "deb https://dl.yarnpkg.com/$(lsb_release -is | tr '[:upper:]' '[:lower:]')/ stable main" | tee /etc/apt/sources.list.d/yarn.list \
    && apt-get update \
    && apt-get -y install --no-install-recommends yarn \
    #
    # Set alternate global install location that both users have rights to access
    && mkdir -p ${NPM_GLOBAL} \
    && chown ${USERNAME}:root ${NPM_GLOBAL} \
    && npm config -g set prefix ${NPM_GLOBAL} \
    && sudo -u ${USERNAME} npm config -g set prefix ${NPM_GLOBAL} \
    #
    # Install NVM to allow installing alternate versions of Node.js as needed
    && mkdir -p ${NVM_DIR} \
    && export NODE_VERSION= \
    && curl -so- https://raw.githubusercontent.com/nvm-sh/nvm/v0.35.3/install.sh | bash 2>&1 \
    && /bin/bash -c "source $NVM_DIR/nvm.sh \
        && nvm alias default system" 2>&1 \
    && echo '[ -s "$NVM_DIR/nvm.sh" ] && \\. "$NVM_DIR/nvm.sh"  && [ -s "$NVM_DIR/bash_completion" ] && \\. "$NVM_DIR/bash_completion"' \ 
    | tee -a /home/${USERNAME}/.bashrc /home/${USERNAME}/.zshrc >> /root/.zshrc \
    && chown ${USER_UID}:${USER_GID} /home/${USERNAME}/.bashrc /home/${USERNAME}/.zshrc \
    && chown -R ${USER_UID}:root ${NVM_DIR} \
    #
    # Handle scenarios where GID/UID is changed
    && echo "if [ \"\$(stat -c '%U' ${NPM_GLOBAL})\" != \"${USERNAME}\" ]; then sudo chown -R ${USER_UID}:root ${NPM_GLOBAL} ${NVM_DIR}; fi" \
    | tee -a /root/.bashrc /root/.zshrc /home/${USERNAME}/.bashrc >> /home/${USERNAME}/.zshrc \
    #
    # Tactically remove imagemagick due to https://security-tracker.debian.org/tracker/CVE-2019-10131
    # Can leave in Dockerfile once upstream base image moves to > 7.0.7-28.
    && apt-get purge -y imagemagick imagemagick-6-common \
    #
    # Install eslint globally
    && sudo -u ${USERNAME} npm install -g eslint

# AWS Tools
RUN \
    # Install AWS CLI v2: https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" \
    && unzip awscliv2.zip \
    && sudo ./aws/install \
    && rm -rf aws awscliv2.zip \
    #
    # Install AWS SAM CLI
    # Install Homebrew before installing aws-sam-cli
    && apt-get -y install ruby build-essential groff \
    && echo | sudo -u ${USERNAME} sh -c "$(curl -fsSL https://raw.githubusercontent.com/Linuxbrew/install/master/install.sh)" \
    && echo 'eval $(/home/linuxbrew/.linuxbrew/bin/brew shellenv)' >> /home/${USERNAME}/.bashrc \
    && eval $(/home/linuxbrew/.linuxbrew/bin/brew shellenv) \
    && brew tap aws/tap \
    && brew install aws-sam-cli \
    #
    # Install AWS ECS CLI
    && sudo curl -o /usr/local/bin/ecs-cli https://s3.cn-north-1.amazonaws.com.cn/amazon-ecs-cli/ecs-cli-linux-amd64-latest \
    && sudo chmod +x /usr/local/bin/ecs-cli \
    #
    # Install AWS AMPLIFY CLI
    && sudo -u ${USERNAME} npm install -g @aws-amplify/cli
    #
    # Install FARGATE CLI
COPY --from=fargatedist /fargate/fargate /usr/local/bin/
    # && curl -L -o fargate-0.3.2-linux-amd64.zip https://github.com/awslabs/fargatecli/releases/download/0.3.2/fargate-0.3.2-linux-amd64.zip \
    # && sudo unzip fargate-0.3.2-linux-amd64.zip -d /usr/local/bin/ \
    # && rm -rf fargate-0.3.2-linux-amd64.zip
    #
    # Install script to
    # 1. automate the emptying of AWS S3 bucket (with object versioning enabled)
    # 2. automate the deletion of iam users
COPY aws-s3-empty-bucket.sh aws-iam-delete-user.sh /usr/local/bin/

# Docker installation
RUN \
    # Install Docker CE CLI (and engine)
    # Fix to version due to bug in 19.03.11 (https://github.com/docker/for-linux/issues/1033)
    apt-get install -y apt-transport-https ca-certificates curl gnupg2 lsb-release \
    && curl -fsSL https://download.docker.com/linux/$(lsb_release -is | tr '[:upper:]' '[:lower:]')/gpg | apt-key add - 2>/dev/null \
    && echo "deb [arch=amd64] https://download.docker.com/linux/$(lsb_release -is | tr '[:upper:]' '[:lower:]') $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list \
    && apt-get update \
    && apt-get install -y containerd.io socat docker-ce-cli=5:19.03.10\* docker-ce=5:19.03.10\* \
    #
    # Install Docker Compose
    && LATEST_COMPOSE_VERSION=$(curl -sSL "https://api.github.com/repos/docker/compose/releases/latest" | grep -o -P '(?<="tag_name": ").+(?=")') \
    && curl -sSL "https://github.com/docker/compose/releases/download/${LATEST_COMPOSE_VERSION}/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose \
    && chmod +x /usr/local/bin/docker-compose


# 3-WAYS TO ENABLE DOCKER IN THE CONTAINER
# https://github.com/Microsoft/vscode-dev-containers/tree/master/containers/docker-in-docker
RUN echo '#!/bin/sh\n\
    if [ "$VSC_DIND" = "root" ] \n\
    then \n\
        # Create docker-root-entrypoint.sh to link host docker socket to container docker socket. This is only for root access.
        # In order to use this script, please ensure that remoteUser is set to root (or removed) in devcontainer.json file.
        sudo ln -s /var/run/docker-host.sock /var/run/docker.sock \n\
        echo Symbolic created at /var/run/docker.sock \n\
    elif [ "$VSC_DIND" = "nonroot" ] \n\
    then \n\
        # Create docker-nonroot-entrypoint.sh to spin up socat, to enable non-root access
        sudo rm -rf /var/run/docker.sock\n\
        ((sudo socat UNIX-LISTEN:/var/run/docker.sock,fork,mode=660,user='${NONROOT_USER}' UNIX-CONNECT:/var/run/docker-host.sock) >> /tmp/vscr-dind-socat.log 2>&1) & > /dev/null\n\
        echo socat: UNIX-LISTEN:/var/run/docker.sock UNIX-CONNECT:/var/run/docker-host.sock \n\
    else \n\
        # Create dockerd-entrypoint.sh to start dockerd in the container, please make sure that non-root users are added to docker group
        # This requires --privileged to be set on devcontainer.json
        ((sudo dockerd --host="unix:///var/run/docker.sock") >> /tmp/vscr-dockerd.log 2>&1) & >/dev/null\n\
        sudo usermod -aG docker '${NONROOT_USER}' \n\
        echo Dockerd started \n\
    fi \n\
    "$@"' >> /usr/local/share/entrypoint.sh \
    && chmod +x /usr/local/share/entrypoint.sh


# Clean up
RUN apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*


# Setting the ENTRYPOINT to [docker-root-entrypoint.sh | docker-nonroot-entrypoint.sh | dockerd-entrypoint.sh]
# will configure docker installation type in container
# when "overrideCommand": false is set in devcontainer.json.
# The script will also execute CMD if you need to alter startup behaviors.
ENTRYPOINT [ "/usr/local/share/entrypoint.sh" ]
CMD [ "sleep", "infinity" ]
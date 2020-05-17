#-------------------------------------------------------------------------------------------------------------
# Copyright (c) Microsoft Corporation. All rights reserved.
# Licensed under the MIT License. See https://go.microsoft.com/fwlink/?linkid=2090316 for license information.
#-------------------------------------------------------------------------------------------------------------

FROM node:14

# The node image includes a non-root user with sudo access. Use the "remoteUser"
# property in devcontainer.json to use it. On Linux, the container user's GID/UIDs
# will be updated to match your local UID/GID (when using the dockerFile property).
# See https://aka.ms/vscode-remote/containers/non-root-user for details.
ARG USERNAME=node
ARG USER_UID=1000
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

# Configure apt and install packages
RUN apt-get update \
    && export DEBIAN_FRONTEND=noninteractive \
    #
    # Verify git, common tools / libs installed, add/modify non-root user, optionally install zsh
    && apt-get -y install --no-install-recommends curl ca-certificates 2>&1 \
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
    && sudo -u ${USERNAME} npm install -g eslint \
    #
    # Install AWS CLI v2: https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html
    && curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" \
    && unzip awscliv2.zip \
    && sudo ./aws/install \
    && rm -rf aws awscliv2.zip \
    #
    # Install Docker CE CLI
    && apt-get install -y apt-transport-https ca-certificates curl gnupg2 lsb-release \
    && curl -fsSL https://download.docker.com/linux/$(lsb_release -is | tr '[:upper:]' '[:lower:]')/gpg | apt-key add - 2>/dev/null \
    && echo "deb [arch=amd64] https://download.docker.com/linux/$(lsb_release -is | tr '[:upper:]' '[:lower:]') $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list \
    && apt-get update \
    && apt-get install -y docker-ce-cli \
    #
    # Install Docker Compose
    && LATEST_COMPOSE_VERSION=$(curl -sSL "https://api.github.com/repos/docker/compose/releases/latest" | grep -o -P '(?<="tag_name": ").+(?=")') \
    && curl -sSL "https://github.com/docker/compose/releases/download/${LATEST_COMPOSE_VERSION}/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose \
    && chmod +x /usr/local/bin/docker-compose
    #
    #
    #
    # ENABLING NON-ROOT ACCESS TO DOCKER IN THE CONTAINER
    # https://github.com/Microsoft/vscode-dev-containers/tree/master/containers/docker-in-docker
    # Default to root only access to the Docker socket, set up non-root init script
    RUN touch /var/run/docker-host.sock \
    && ln -s /var/run/docker-host.sock /var/run/docker.sock \
    && apt-get update \
    && apt-get -y install socat
    #
    # Create docker-init.sh to spin up socat
    RUN echo '#!/bin/sh\n\
    sudo rm -rf /var/run/docker.sock\n\
    ((sudo socat UNIX-LISTEN:/var/run/docker.sock,fork,mode=660,user='${NONROOT_USER}' UNIX-CONNECT:/var/run/docker-host.sock) 2>&1 >> /tmp/vscr-dind-socat.log) & > /dev/null\n\
    "$@"' >> /usr/local/share/docker-init.sh \
    && chmod +x /usr/local/share/docker-init.sh
    #
    #
    #
    #
    # INSTALL AWS SAM CLI
    # Install Homebrew
    RUN apt-get -y install ruby \
    && sudo apt-get install build-essential \
    && echo | sudo -u node sh -c "$(curl -fsSL https://raw.githubusercontent.com/Linuxbrew/install/master/install.sh)" \
    && echo 'eval $(/home/linuxbrew/.linuxbrew/bin/brew shellenv)' >> /home/${USERNAME}/.bashrc \
    && eval $(/home/linuxbrew/.linuxbrew/bin/brew shellenv) \
    && brew tap aws/tap \
    && brew install aws-sam-cli
    #
    # Clean up
    RUN apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*
    #
    #
    # Setting the ENTRYPOINT to docker-init.sh will configure non-root access to
    # the Docker socket if "overrideCommand": false is set in devcontainer.json.
    # The script will also execute CMD if you need to alter startup behaviors.
    ENTRYPOINT [ "/usr/local/share/docker-init.sh" ]
    CMD [ "sleep", "infinity" ]


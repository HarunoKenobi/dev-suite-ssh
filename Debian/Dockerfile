# syntax=docker/dockerfile:1
FROM debian:12-slim

ARG DOTNET_VERSION=10.0
ARG NODE_MAJOR=22
ARG USERNAME=dev

ENV DEBIAN_FRONTEND=noninteractive \
    TZ=Etc/UTC \
    DOTNET_CLI_TELEMETRY_OPTOUT=1 \
    DOTNET_NOLOGO=1 \
    NPM_CONFIG_PREFIX=/home/dev/.npm-global

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# SSH, VS Code Remote SSH prerequisites, and common development utilities.
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        bash \
        ca-certificates \
        curl \
        git \
        gnupg \
        locales \
        openssh-server \
        sudo \
        tar \
        tzdata \
        wget \
        zip \
    && rm -rf /var/lib/apt/lists/* \
    && sed -i 's/^# \(en_US.UTF-8 UTF-8\)/\1/' /etc/locale.gen \
    && locale-gen

# Microsoft .NET SDK repository for Debian 12.
RUN curl -fsSL https://packages.microsoft.com/config/debian/12/packages-microsoft-prod.deb -o /tmp/packages-microsoft-prod.deb \
    && dpkg -i /tmp/packages-microsoft-prod.deb \
    && rm /tmp/packages-microsoft-prod.deb \
    && apt-get update \
    && apt-get install -y --no-install-recommends "dotnet-sdk-${DOTNET_VERSION}" \
    && rm -rf /var/lib/apt/lists/*

# Node.js LTS repository.
RUN mkdir -p /etc/apt/keyrings \
    && curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key \
        | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg \
    && echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_${NODE_MAJOR}.x nodistro main" \
        > /etc/apt/sources.list.d/nodesource.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends nodejs \
    && rm -rf /var/lib/apt/lists/*

# Develop through SSH as a non-root user. Let Debian select free IDs.
RUN groupadd "${USERNAME}" \
    && useradd --gid "${USERNAME}" --create-home --shell /bin/bash "${USERNAME}" \
    # Unlock the account for public-key SSH; PasswordAuthentication remains disabled in sshd.
    && passwd --delete "${USERNAME}" \
    && echo "${USERNAME} ALL=(ALL) NOPASSWD:ALL" > "/etc/sudoers.d/${USERNAME}" \
    && chmod 0440 "/etc/sudoers.d/${USERNAME}" \
    && install -d -m 0700 -o "${USERNAME}" -g "${USERNAME}" "/home/${USERNAME}/.ssh" \
    && install -d -m 0755 -o "${USERNAME}" -g "${USERNAME}" "/home/${USERNAME}/workspace" \
    && install -d -m 0755 -o "${USERNAME}" -g "${USERNAME}" "/home/${USERNAME}/.npm-global"

# Install Claude Code in the developer account: no sudo/npm permission issues.
USER ${USERNAME}
ENV HOME=/home/${USERNAME} \
    PATH=/home/${USERNAME}/.npm-global/bin:${PATH}
RUN npm install --global @anthropic-ai/claude-code \
    && claude --version

USER root

# sshd supplies its own login PATH; expose Claude Code in a system PATH directory.
RUN ln -s /home/${USERNAME}/.npm-global/bin/claude /usr/local/bin/claude

# SSH is key-only. Provide authorized_keys at runtime through the compose bind mount.
RUN rm -f /etc/ssh/ssh_host_* \
    && printf '%s\n' \
        'PermitRootLogin no' \
        'PasswordAuthentication no' \
        'KbdInteractiveAuthentication no' \
        'PubkeyAuthentication yes' \
        'AllowUsers dev' \
        'X11Forwarding no' \
        'UsePAM no' \
        'ClientAliveInterval 120' \
        'ClientAliveCountMax 2' \
        > /etc/ssh/sshd_config.d/dev-container.conf \
    && mkdir -p /run/sshd

WORKDIR /home/${USERNAME}/workspace
EXPOSE 22

# Create unique SSH host keys when each container starts.
CMD ["/bin/bash", "-lc", "ssh-keygen -A && exec /usr/sbin/sshd -D -e"]

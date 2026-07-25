FROM mcr.microsoft.com/devcontainers/base:ubuntu-24.04
#FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    curl \
    ca-certificates \
    sudo \
    openssh-client \
    zsh \
    locales \
    unzip \
    build-essential \
    && rm -rf /var/lib/apt/lists/*


#ARG USERNAME=vscode
#ARG USER_UID=1000
#ARG USER_GID=$USER_UID
#
#RUN groupadd --gid $USER_GID $USERNAME \
#    && useradd --uid $USER_UID --gid $USER_GID -m $USERNAME \
#    && echo $USERNAME ALL=\(ALL\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME \
#    && chmod 0440 /etc/sudoers.d/$USERNAME \
#    && curl https://mise.run | MISE_INSTALL_PATH=/usr/local/bin/mise sh
RUN curl https://mise.run | MISE_INSTALL_PATH=/usr/local/bin/mise sh

# 3. Switch context to the user for environment configuration
USER $USERNAME
RUN echo 'eval "$(mise activate bash)"' >> ~/.bashrc \
    && echo 'eval "$(mise activate zsh)"' >> ~/.zshrc 

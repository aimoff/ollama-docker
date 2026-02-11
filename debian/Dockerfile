ARG OS_VERSION=trixie

FROM debian:${OS_VERSION}

RUN apt-get update \
  && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
  sudo \
  curl \
  zstd \
  pciutils \
  lshw \
  supervisor \
  locales \
  git \
  vim-nox \
  less \
  python3-pip \
  python3-venv \
  python3-argcomplete

# Grant members of 'sudo' group passwordless privileges
COPY sudo-nopasswd /etc/sudoers.d/sudo-nopasswd

# Create render group and ollama user
ARG RENDER_GID
RUN groupadd -g "$RENDER_GID" render; \
    useradd --system -U -G video,render --shell /bin/false --create-home ollama 

# Install Ollama
RUN curl -fsSL https://ollama.com/install.sh | sh && ldconfig

# Create user $USER_NAME as member of sudo group
ARG UID
ARG USER_NAME=user
RUN UID_ADD=; [ -z "$UID" ] || UID_ADD="-u $UID"; \
    useradd $UID_ADD -G sudo,video,render --shell /bin/bash --create-home $USER_NAME

# Locale setting
ARG LANG=C.UTF-8
RUN sed -i "/$LANG/s/^# //" /etc/locale.gen && locale-gen

#RUN DEBIAN_FRONTEND=noninteractive apt-get purge --autoremove -y \
#  zstd \
#  pciutils \
#  lshw
RUN apt-get clean \
  && rm -rf /var/lib/apt/lists/*

COPY supervisord.conf /etc/supervisor/conf.d/ollama.conf

EXPOSE 11434
USER $USER_NAME
WORKDIR /home/$USER_NAME
CMD ["/usr/bin/sudo", "/usr/bin/supervisord", "--nodaemon"]

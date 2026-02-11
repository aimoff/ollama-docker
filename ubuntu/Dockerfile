FROM ollama/ollama:rocm

RUN apt-get update \
  && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
  sudo \
  curl \
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

# Create user $USER_NAME as member of sudo group
ARG UID=1000
ARG RENDER_GID
ARG USER_NAME=user
RUN groupadd -g "$RENDER_GID" render; \
    useradd --system -U -G video,render --shell /bin/false --create-home ollama; \
    if [ -n "$UID" ]; then \
        if [ "$UID" -eq 1000 ]; then \
            if [ "$USER_NAME" != ubuntu ]; then \
                usermod -l $USER_NAME ubuntu; \
                usermod -md /home/$USER_NAME $USER_NAME; \
            fi; \
            usermod -aG render $USER_NAME; \
        else \
            useradd -u $UID -G sudo,video,render --shell /bin/bash --create-home $USER_NAME; \
        fi; \
    else \
        useradd -G sudo,video,render --shell /bin/bash --create-home $USER_NAME; \
    fi

# Locale setting
ARG LANG=C.UTF-8
RUN sed -i "/$LANG/s/^# //" /etc/locale.gen && locale-gen

RUN apt-get clean \
  && rm -rf /var/lib/apt/lists/*

COPY supervisord.conf /etc/supervisor/conf.d/ollama.conf

EXPOSE 11434
USER $USER_NAME
WORKDIR /home/$USER_NAME
ENTRYPOINT []
CMD ["/usr/bin/sudo", "/usr/bin/supervisord", "--nodaemon"]

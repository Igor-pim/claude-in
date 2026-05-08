FROM debian:bookworm-slim

ARG USER_UID=1000
ARG USER_GID=1000

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git jq less ripgrep gnupg sudo \
      sox libsox-fmt-pulse libsox-fmt-alsa pulseaudio-utils \
      alsa-utils libasound2-plugins \
    && rm -rf /var/lib/apt/lists/*

# Route ALSA default PCM through PulseAudio so `arecord` and `sox -d` work
# without explicit `-t pulseaudio`. Claude Code's voice mode shells out to
# either tool; with this + PULSE_SERVER=unix:/mnt/wslg/PulseServer it picks
# up the WSLg microphone on Windows 11.
RUN printf 'pcm.!default { type pulse }\nctl.!default { type pulse }\n' > /etc/asound.conf

# Node.js 20 (for Claude Code)
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y --no-install-recommends nodejs \
    && rm -rf /var/lib/apt/lists/*

# Docker CLI + compose plugin (client only, no daemon)
RUN install -m 0755 -d /etc/apt/keyrings \
    && curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc \
    && chmod a+r /etc/apt/keyrings/docker.asc \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian bookworm stable" \
       > /etc/apt/sources.list.d/docker.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends docker-ce-cli docker-compose-plugin \
    && rm -rf /var/lib/apt/lists/*

RUN npm install -g @anthropic-ai/claude-code

RUN groupadd -g ${USER_GID} dev \
    && useradd -m -u ${USER_UID} -g ${USER_GID} -s /bin/bash dev

USER dev
WORKDIR /workspace
CMD ["sleep", "infinity"]

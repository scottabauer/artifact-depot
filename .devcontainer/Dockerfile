# Dev container for working on Artifact Depot.
#
# Split into three clearly labeled sections so the container layout can
# be kept in lockstep with our sibling projects:
#   * COMMONIZED FOUNDATION -- base image, apt sources, base packages,
#     and the CLIs every project installs the same way (rust, reuse,
#     hadolint, glab, claude). All installed system-wide, as root, into
#     /usr/local so they are on PATH for whatever UID the dev user ends
#     up with -- never trapped in /root.
#   * PROJECT-SPECIFIC -- Artifact Depot's extras (DynamoDB Local, helm,
#     ruby/jekyll, docker-compose, playwright).
#   * COMMONIZED FOOTER -- dev user creation, USER, WORKDIR. Must come
#     last so /workspace/artifact-depot is created owned by the dev user
#     instead of root.

# =============================================================================
# COMMONIZED FOUNDATION -- base image, apt sources, base packages, then
# the CLIs every sibling project installs the same way, all system-wide.
# =============================================================================

ARG DOCKER_MIRROR=
FROM ${DOCKER_MIRROR}ubuntu:24.04

ARG APT_MIRROR=
ENV DEBIAN_FRONTEND=noninteractive

# Narrow the sources to the components and suites we actually consume
# (main + universe; no backports). This also lets APT_MIRROR point at
# mirrors that do not expose restricted, multiverse, or noble-backports.
RUN sed -Ei \
        -e 's/^Components:.*/Components: main universe/' \
        -e 's/ noble-backports//' \
        /etc/apt/sources.list.d/ubuntu.sources \
    && if [ -n "$APT_MIRROR" ]; then \
         sed -Ei "s#https?://(archive|security)\.ubuntu\.com/ubuntu/?#http://${APT_MIRROR}/#g" \
           /etc/apt/sources.list.d/ubuntu.sources; \
       fi

# Unminimize, then install everything from apt in a single pass.
RUN apt-get -y update && \
    apt-get -y --no-install-recommends upgrade && \
    apt-get -y --no-install-recommends install unminimize && \
    echo y | unminimize && \
    apt-get -y --no-install-recommends install \
      openssh-client git npm curl wget vim make build-essential sudo \
      bash-completion less unzip ripgrep shellcheck file \
      iproute2 python3 pipx default-jre-headless mold gh jq skopeo \
      nginx-light openssl && \
    rm -rf /var/lib/apt/lists/*

# Detect target arch once for the release downloads further down. Two
# naming conventions appear in those assets, so capture the Debian-style
# name (amd64/arm64) here for Helm and glab; AWS CLI and docker-compose
# consume `uname -m` (x86_64/aarch64) directly, and hadolint maps
# aarch64->arm64 inline.
RUN case "$(uname -m)" in \
        x86_64)  echo amd64 ;; \
        aarch64) echo arm64 ;; \
        *) echo "unsupported architecture: $(uname -m)" >&2; exit 1 ;; \
    esac > /tmp/arch_deb

# Free up UID 1000 so the dev user (created in the footer) can claim it
# cleanly. updateRemoteUserUID then patches that user to match the host
# UID at container start.
RUN userdel -r ubuntu || true

# Rust toolchain + cargo workspace linters, installed system-wide so
# cargo/rustc/clippy/rustfmt and the linters land in /usr/local/cargo and
# are on PATH for every user rather than trapped in /root/.cargo. The
# toolchain dirs are left world-writable so the dev user can populate the
# registry/git cache during builds (mirrors the official `rust` image).
ENV RUSTUP_HOME=/usr/local/rustup \
    CARGO_HOME=/usr/local/cargo \
    PATH=/usr/local/cargo/bin:$PATH
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y \
        --default-toolchain 1.94.1 --no-modify-path && \
    cargo install cargo-deny --locked && \
    cargo install cargo-about --locked --features cli && \
    chmod -R a+rwX "$RUSTUP_HOME" "$CARGO_HOME"

# reuse (REUSE 3.x license linter). Ubuntu 24.04 ships reuse 2.1 via apt,
# which predates .gitignore support, so install via pipx into a
# system-wide location reachable by every user. reuse needs an encoding
# detector at runtime; the `file` apt package above provides the libmagic
# backend (without it, every `reuse lint` aborts with NoEncodingModuleError).
RUN PIPX_HOME=/usr/local/pipx PIPX_BIN_DIR=/usr/local/bin pipx install reuse && \
    chmod -R a+rX /usr/local/pipx

# hadolint (Dockerfile linter; backs the exiasr.hadolint VSCode extension).
ARG HADOLINT_VERSION=2.12.0
RUN ARCH="$(uname -m | sed 's/aarch64/arm64/')" && \
    curl -fsSLo /usr/local/bin/hadolint \
        "https://github.com/hadolint/hadolint/releases/download/v${HADOLINT_VERSION}/hadolint-Linux-${ARCH}" && \
    chmod 0755 /usr/local/bin/hadolint

# glab (GitLab CLI). Latest release pulled straight from gitlab.com,
# arch-matched. Used for talking to Quantum's GitLab (gitlab.mdh.quantum.com)
# from inside the container; reads a token from the environment (see
# ~/.dev_aliases forwarding in devcontainer.json).
RUN ARCH="$(cat /tmp/arch_deb)" && \
    GLAB_TAG="$(curl -fsSL 'https://gitlab.com/api/v4/projects/gitlab-org%2Fcli/releases/permalink/latest' \
        | grep -o '"tag_name":"[^"]*"' | sed 's/"tag_name":"//;s/"//')" && \
    GLAB_VERSION="${GLAB_TAG#v}" && \
    mkdir -p /tmp/glab && \
    curl -fsSL "https://gitlab.com/gitlab-org/cli/-/releases/${GLAB_TAG}/downloads/glab_${GLAB_VERSION}_linux_${ARCH}.tar.gz" \
        | tar -xz -C /tmp/glab && \
    mv /tmp/glab/bin/glab /usr/local/bin/glab && \
    rm -rf /tmp/glab

# Claude Code native client. Installed system-wide so `claude` is on PATH
# for every user from container start -- no per-user postCreate install,
# no window reload for the VSCode Claude extension to find it. The
# installer hardcodes $HOME/.local/bin/claude (and refuses to be
# redirected), so we run it as root and relocate the resolved binary to
# /usr/local/bin, then drop root's copy.
RUN curl -fsSL https://claude.ai/install.sh | bash -s -- stable && \
    install -m 0755 "$(readlink -f /root/.local/bin/claude)" /usr/local/bin/claude && \
    rm -rf /root/.local /root/.claude

# =============================================================================
# PROJECT-SPECIFIC -- Artifact Depot's extras on top of the foundation.
# =============================================================================

# AWS CLI v2, for operating the depot instances we run on AWS (S3 blob
# store, DynamoDB KV store). The bundled installer needs `unzip` (pulled
# in by the base apt layer above). Installs into /usr/local/aws-cli with
# the `aws` symlink in /usr/local/bin so it is on PATH for every user.
RUN curl -fsSL "https://awscli.amazonaws.com/awscli-exe-linux-$(uname -m).zip" -o /tmp/awscliv2.zip && \
    unzip -q /tmp/awscliv2.zip -d /tmp && \
    /tmp/aws/install && \
    rm -rf /tmp/awscliv2.zip /tmp/aws

# DynamoDB Local (for integration testing)
ENV DYNAMODB_LOCAL_DIR=/usr/local/lib/dynamodb-local
RUN mkdir -p "$DYNAMODB_LOCAL_DIR" && \
    wget -qO /tmp/dynamodb_local.tar.gz "https://d1ni2b6xgvw0s0.cloudfront.net/v2.x/dynamodb_local_latest.tar.gz" && \
    tar xzf /tmp/dynamodb_local.tar.gz -C "$DYNAMODB_LOCAL_DIR" && \
    rm -f /tmp/dynamodb_local.tar.gz

# Helm (for linting and templating the depot Helm chart)
ENV HELM_VERSION=v3.16.2
RUN ARCH="$(cat /tmp/arch_deb)" && \
    curl -fsSL "https://get.helm.sh/helm-${HELM_VERSION}-linux-${ARCH}.tar.gz" -o /tmp/helm.tar.gz && \
    tar xzf /tmp/helm.tar.gz -C /tmp && \
    mv "/tmp/linux-${ARCH}/helm" /usr/local/bin/helm && \
    rm -rf /tmp/helm.tar.gz "/tmp/linux-${ARCH}"

# Ruby + Bundler for the Jekyll docs site (just-the-docs theme).
# Ubuntu 24.04 ships Ruby 3.2 in `ruby-full`, which is sufficient for
# Jekyll 4 + just-the-docs 0.10.  The native-gem build tools (libffi-dev,
# zlib1g-dev, libyaml-dev) are needed because sass-embedded compiles native
# extensions during `bundle install`.
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        ruby-full ruby-bundler ruby-dev \
        libffi-dev zlib1g-dev libyaml-dev && \
    rm -rf /var/lib/apt/lists/*

# docker-compose v1 (standalone upstream binary) for docker/standalone +
# docker/distributed + the observability-screenshots harness.  The host's
# Docker Engine daemon is reached via the mounted /var/run/docker.sock.
#
# We grab the PyInstaller-bundled Linux release directly from GitHub
# rather than the Ubuntu `docker-compose` apt package, which on 24.04 is
# broken: it imports `distutils` at load time and Python 3.12 dropped
# `distutils` from stdlib.
RUN curl -fsSL -o /usr/local/bin/docker-compose \
        "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-Linux-$(uname -m)" && \
    chmod +x /usr/local/bin/docker-compose

# Chrome-for-Testing for the UI end-to-end tests and the @playwright/mcp
# MCP server that .mcp.json launches via `npx -y @playwright/mcp@latest
# --browser chromium`. We install:
#   * OS deps via `playwright install-deps chromium` -- the same C
#     libraries and fonts chrome-for-testing needs to launch. Lets
#     Playwright own that dependency list rather than hand-maintaining it
#     in the apt layer above. The base layer cleared the apt lists, so we
#     refresh them first and clear them again after.
#   * The chrome-for-testing browser itself via @playwright/mcp's own
#     `install-browser` command (despite the name it really is
#     chrome-for-testing, the build the MCP looks for at runtime -- not
#     Playwright's vendored chromium) so the MCP finds it offline rather
#     than downloading on first launch.
# Installed into a system-wide PLAYWRIGHT_BROWSERS_PATH rather than a
# per-user cache, left sticky world-writable so the dev user can create
# its per-launch profile dirs and a newer MCP can drop newer browser
# builds alongside.
ENV PLAYWRIGHT_BROWSERS_PATH=/opt/playwright-browsers
RUN apt-get -y update && \
    mkdir -p "${PLAYWRIGHT_BROWSERS_PATH}" && \
    npx -y playwright install-deps chromium && \
    npx -y @playwright/mcp@latest install-browser chrome-for-testing && \
    chmod -R a+rwX "${PLAYWRIGHT_BROWSERS_PATH}" && \
    chmod 1777 "${PLAYWRIGHT_BROWSERS_PATH}" && \
    rm -rf /var/lib/apt/lists/*

# =============================================================================
# COMMONIZED FOOTER -- dev user, USER, WORKDIR. Must come last so
# /workspace/artifact-depot is created owned by the dev user instead of
# root. Without the WORKDIR-after-USER trick, docker auto-creates it as
# root:root when it mounts the workspace in.
#
# sudoers rule goes to a fixed filename, not ${USERNAME}: sudo ignores
# sudoers.d files whose name contains a dot (e.g. first.last).
# =============================================================================

ARG USERNAME
RUN useradd -m -s /bin/bash "${USERNAME}" && \
    echo "${USERNAME} ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/devuser && \
    chmod 0440 /etc/sudoers.d/devuser

USER ${USERNAME}
WORKDIR /workspace/artifact-depot

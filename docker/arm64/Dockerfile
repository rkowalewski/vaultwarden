# This file was generated using a Jinja2 template.
# Please make your changes in `Dockerfile.j2` and then `make` the individual Dockerfiles.

# Using multistage build:
# 	https://docs.docker.com/develop/develop-images/multistage-build/
# 	https://whitfin.io/speeding-up-rust-docker-builds/
####################### VAULT BUILD IMAGE  #######################
# The web-vault digest specifies a particular web-vault build on Docker Hub.
# Using the digest instead of the tag name provides better security,
# as the digest of an image is immutable, whereas a tag name can later
# be changed to point to a malicious image.
#
# To verify the current digest for a given tag name:
# - From https://hub.docker.com/r/vaultwarden/web-vault/tags,
#   click the tag name to view the digest of the image it currently points to.
# - From the command line:
#     $ docker pull vaultwarden/web-vault:v2.19.0d
#     $ docker image inspect --format "{{.RepoDigests}}" vaultwarden/web-vault:v2.19.0d
#     [vaultwarden/web-vault@sha256:a7bd6bc4db33bd45f723c4b1ac90918b7f80204560683cfc8efd9efd03a9b233]
#
# - Conversely, to get the tag name from the digest:
#     $ docker image inspect --format "{{.RepoTags}}" vaultwarden/web-vault@sha256:a7bd6bc4db33bd45f723c4b1ac90918b7f80204560683cfc8efd9efd03a9b233
#     [vaultwarden/web-vault:v2.19.0d]
#
FROM vaultwarden/web-vault@sha256:a7bd6bc4db33bd45f723c4b1ac90918b7f80204560683cfc8efd9efd03a9b233 as vault

########################## BUILD IMAGE  ##########################
FROM rust:1.51 as build

# Debian-based builds support multidb
ARG DB=sqlite,mysql,postgresql

# Build time options to avoid dpkg warnings and help with reproducible builds.
ENV DEBIAN_FRONTEND=noninteractive LANG=C.UTF-8 TZ=UTC TERM=xterm-256color

# Don't download rust docs
RUN rustup set profile minimal

# Install required build libs for arm64 architecture.
# To compile both mysql and postgresql we need some extra packages for both host arch and target arch
RUN sed 's/^deb/deb-src/' /etc/apt/sources.list > \
        /etc/apt/sources.list.d/deb-src.list \
    && dpkg --add-architecture arm64 \
    && apt-get update \
    && apt-get install -y \
        --no-install-recommends \
        libssl-dev:arm64 \
        libc6-dev:arm64 \
        libpq5:arm64 \
        libpq-dev \
        libmariadb-dev:arm64 \
        libmariadb-dev-compat:arm64

RUN apt-get update \
    && apt-get install -y \
        --no-install-recommends \
        gcc-aarch64-linux-gnu \
    && mkdir -p ~/.cargo \
    && echo '[target.aarch64-unknown-linux-gnu]' >> ~/.cargo/config \
    && echo 'linker = "aarch64-linux-gnu-gcc"' >> ~/.cargo/config \
    && echo 'rustflags = ["-L/usr/lib/aarch64-linux-gnu"]' >> ~/.cargo/config

ENV CARGO_HOME "/root/.cargo"
ENV USER "root"

# Creates a dummy project used to grab dependencies
RUN USER=root cargo new --bin /app
WORKDIR /app

# Copies over *only* your manifests and build files
COPY ./Cargo.* ./
COPY ./rust-toolchain ./rust-toolchain
COPY ./build.rs ./build.rs

# NOTE: This should be the last apt-get/dpkg for this stage, since after this it will fail because of broken dependencies.
# For Diesel-RS migrations_macros to compile with MySQL/MariaDB we need to do some magic.
# We at least need libmariadb3:amd64 installed for the x86_64 version of libmariadb.so (client)
# We also need the libmariadb-dev-compat:amd64 but it can not be installed together with the :arm64 version.
# What we can do is a force install, because nothing important is overlapping each other.
RUN apt-get install -y --no-install-recommends libmariadb3:amd64 && \
    apt-get download libmariadb-dev-compat:amd64 && \
    dpkg --force-all -i ./libmariadb-dev-compat*.deb && \
    rm -rvf ./libmariadb-dev-compat*.deb

# For Diesel-RS migrations_macros to compile with PostgreSQL we need to do some magic.
# The libpq5:arm64 package seems to not provide a symlink to libpq.so.5 with the name libpq.so.
# This is only provided by the libpq-dev package which can't be installed for both arch at the same time.
# Without this specific file the ld command will fail and compilation fails with it.
RUN ln -sfnr /usr/lib/aarch64-linux-gnu/libpq.so.5 /usr/lib/aarch64-linux-gnu/libpq.so

ENV CC_aarch64_unknown_linux_gnu="/usr/bin/aarch64-linux-gnu-gcc"
ENV CROSS_COMPILE="1"
ENV OPENSSL_INCLUDE_DIR="/usr/include/aarch64-linux-gnu"
ENV OPENSSL_LIB_DIR="/usr/lib/aarch64-linux-gnu"
RUN rustup target add aarch64-unknown-linux-gnu

# Builds your dependencies and removes the
# dummy project, except the target folder
# This folder contains the compiled dependencies
RUN cargo build --features ${DB} --release --target=aarch64-unknown-linux-gnu
RUN find . -not -path "./target*" -delete

# Copies the complete project
# To avoid copying unneeded files, use .dockerignore
COPY . .

# Make sure that we actually build the project
RUN touch src/main.rs

# Builds again, this time it'll just be
# your actual source files being built
RUN cargo build --features ${DB} --release --target=aarch64-unknown-linux-gnu

######################## RUNTIME IMAGE  ########################
# Create a new stage with a minimal image
# because we already have a binary built
FROM balenalib/aarch64-debian:buster

ENV ROCKET_ENV "staging"
ENV ROCKET_PORT=80
ENV ROCKET_WORKERS=10

RUN [ "cross-build-start" ]

# Install needed libraries
RUN apt-get update && apt-get install -y \
    --no-install-recommends \
    openssl \
    ca-certificates \
    curl \
    dumb-init \
    libmariadb-dev-compat \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir /data

RUN [ "cross-build-end" ]

VOLUME /data
EXPOSE 80
EXPOSE 3012

# Copies the files from the context (Rocket.toml file and web-vault)
# and the binary from the "build" stage to the current stage
WORKDIR /
COPY Rocket.toml .
COPY --from=vault /web-vault ./web-vault
COPY --from=build /app/target/aarch64-unknown-linux-gnu/release/vaultwarden .

COPY docker/healthcheck.sh /healthcheck.sh
COPY docker/start.sh /start.sh

HEALTHCHECK --interval=60s --timeout=10s CMD ["/healthcheck.sh"]

# Configures the startup!
ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["/start.sh"]

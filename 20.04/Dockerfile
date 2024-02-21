# syntax=docker/dockerfile:1.4
FROM ubuntu:20.04 AS base

# This is a automatic arg set when using the buildkit backend
# we use it as a component in cache mounts and use it for the
# architecture of the package
ARG TARGETARCH
ENV TARGETARCH=$TARGETARCH

# The next section are build arguments that are required to build
# The ruby version is expected to be in Major.Minor.Patch format
ARG RUBY_VERSION
ENV RUBY_VERSION=$RUBY_VERSION

ARG RUBY_SHA256
ENV RUBY_SHA256=$RUBY_SHA256

# Build directories
ENV BUILDBASE=/buildroot
ENV SOURCEBASE=$BUILDBASE/src
ENV INSTALLBASE=$BUILDBASE/install
ENV PACKAGEBASE=$BUILDBASE/pkg

# Set the default shell so it can be reused in the stages that inherit from base
SHELL ["/bin/bash", "-xe", "-o", "pipefail", "-c"]

WORKDIR /

RUN rm -f /etc/apt/apt.conf.d/docker-clean; echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache
RUN --mount=type=cache,id=apt-cache-$TARGETARCH,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,id=apt-lib-$TARGETARCH,target=/var/lib/apt,sharing=locked \
      apt-get update

# Make build directories
RUN mkdir -p $BUILDBASE \
      && mkdir -p $SOURCEBASE \
      && mkdir -p $INSTALLBASE \
      && mkdir -p $PACKAGEBASE

#######################
# Pull all the sources
#######################
FROM base AS sources

WORKDIR /tmp

RUN --mount=type=cache,id=apt-cache-$TARGETARCH,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,id=apt-lib-$TARGETARCH,target=/var/lib/apt,sharing=locked \
      rubyMajorMinor=$(echo "$RUBY_VERSION" | cut -d '.' -f1,2) \
      && apt-get update && apt-get install -y --no-install-recommends curl ca-certificates \
      && curl "https://cache.ruby-lang.org/pub/ruby/$rubyMajorMinor/ruby-$RUBY_VERSION.tar.gz" -o "ruby-$RUBY_VERSION.tar.gz" \
      && echo "$RUBY_SHA256 ruby-$RUBY_VERSION.tar.gz" | sha256sum --strict -c \
      && tar -xzf "ruby-$RUBY_VERSION.tar.gz" -C "$SOURCEBASE" --strip-components=1 \
      && rm "ruby-$RUBY_VERSION.tar.gz"

#####################################
# System dependencies and ruby build
#####################################
FROM base AS build

WORKDIR $SOURCEBASE

RUN --mount=type=cache,id=apt-cache-$TARGETARCH,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,id=apt-lib-$TARGETARCH,target=/var/lib/apt,sharing=locked \
      buildDeps=("build-essential" \
                 "pkg-config" \
                 "libffi-dev" \
                 "libgmp-dev" \
                 "libreadline-dev" \
                 "libssl-dev" \
                 "libyaml-dev" \
                 "zlib1g-dev"); \
      # ruby 3.1+ does not have gdbm or dbm as default gems, so these dependencies arent needed
      if dpkg --compare-versions "$RUBY_VERSION" lt "3.1"; then \
        buildDeps+=("libgdbm-dev" "libgdbm-compat-dev"); \
      fi \
      # ruby 3.2+ needs rust to build yjit support
      && if dpkg --compare-versions "$RUBY_VERSION" ge "3.2"; then\
        buildDeps+=("rustc"); \
      fi \
      && apt-get update && apt-get install -y --no-install-recommends "${buildDeps[@]}"

COPY --link --from=sources $SOURCEBASE .

RUN ./configure --prefix=/usr --disable-install-doc \
      && make -j"$(nproc)" \
      && make install DESTDIR="$INSTALLBASE"

############
# Packaging
############
FROM base AS package

RUN --mount=type=cache,id=apt-cache-$TARGETARCH,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,id=apt-lib-$TARGETARCH,target=/var/lib/apt,sharing=locked \
      apt-get update && apt-get install -y --no-install-recommends gettext

WORKDIR $PACKAGEBASE

COPY --link --from=build $INSTALLBASE $INSTALLBASE
COPY DEBIAN $INSTALLBASE/DEBIAN

# Setup control file
RUN runtimeDeps="libffi7, libgmp10, libreadline8, libssl1.1, libyaml-0-2, zlib1g"; \
      # ruby 3.1 does not have gdbm or dbm as default gems, so these dependencies arent needed
      if dpkg --compare-versions "$RUBY_VERSION" lt "3.1"; then \
        runtimeDeps="${runtimeDeps}, libgdbm6, libgdbm-compat4"; \
      fi \
      && RUNTIME_DEPS="$runtimeDeps" \
         envsubst < "$INSTALLBASE/DEBIAN/control.template" > "$INSTALLBASE/DEBIAN/control" \
      && rm "$INSTALLBASE/DEBIAN/control.template"

# The last three run steps could be joined into one but they each do quite different things
# so I have kept them split up as an easier way to document them
# Lastly we don't need to optimize for reduced size because we don't push this stage to a registry

# Create package
RUN dpkg-deb --build "$INSTALLBASE" .

# Create symlink so package can be referenced without the architecture when installed
RUN ln -s "ruby-full_${RUBY_VERSION}_${TARGETARCH}.deb" "ruby-full_${RUBY_VERSION}.deb"

# Output sha256 sums of the packages so they can be validated later if needed
RUN sha256sum ./*.deb

FROM scratch AS final

WORKDIR /

COPY --from=package /buildroot/pkg/ .

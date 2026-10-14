# syntax=docker/dockerfile:1.7
#
# net-snmp commit-image (SMALL form) — builds ON TOP of the org C/C++ base image
# (ghcr.io/mayhemheroes/base). The base provides debian + clang/llvm + libFuzzer + the build
# identities + the build-contract ENV (CC/CXX/SANITIZER_FLAGS/LIB_FUZZING_ENGINE/SRC/
# STANDALONE_FUZZ_MAIN); this only adds the per-commit net-snmp build.
#
# Build (from the fork root):
#   DOCKER_BUILDKIT=1 docker build -f mayhem/Dockerfile \
#       --build-arg COMMIT=$(git rev-parse HEAD) \
#       [--build-arg SOURCE_DATE_EPOCH=$(date +%s)] \
#       -t net-snmp-commit:test .

FROM ghcr.io/savantenvs/base:20260621

ARG COMMIT
ARG REPO_URL
# SOURCE_DATE_EPOCH stays an ARG (available to build.sh); never bake it as ENV — an empty value
# breaks clang. build.sh unsets it when empty.
ARG SOURCE_DATE_EPOCH

# Sanitizer override: the base exports SANITIZER_FLAGS (ASan+UBSan, halting) as the default.
# Declare it WITHOUT a default so an unset arg lets the base ENV flow through; --build-arg overrides.
ARG SANITIZER_FLAGS
ARG MAYHEM_JOBS=

# net-snmp ./configure needs OpenSSL dev headers + pkg-config + perl (already in the base). Code at this
# BASE (2021) also requires libnl-3 for the Linux interface table (if-mib/data_access/interface_linux.c),
# which the base lacks. Install anything missing as root, then drop back to the non-root build user.
USER root
RUN if ! { command -v perl && command -v pkg-config && [ -f /usr/include/openssl/opensslv.h ] \
           && [ -f /usr/include/libnl3/netlink/cache.h ] && [ -f /usr/include/libnl3/netlink/route/link.h ]; } >/dev/null 2>&1; then \
      [ -n "$SOURCE_DATE_EPOCH" ] && repro-sources-list.sh || true; \
      apt-get update && apt-get install -yq --no-install-recommends libssl-dev pkg-config perl \
          libnl-3-dev libnl-route-3-dev libnl-genl-3-dev \
      && rm -rf /var/lib/apt/lists/*; \
    fi
USER mayhem

# Source enters from the build CONTEXT (the CI checkout): the FULL repo (incl. .git history) is
# copied to /mayhem as `mayhem`; build.sh configures + builds in place.
WORKDIR /mayhem
COPY --chown=2000:2000 . /mayhem

RUN bash mayhem/build.sh

# Run the golden BER/PDU-parse oracle at build time (a failing oracle fails the image build).
RUN if [ -f mayhem/test.sh ]; then bash mayhem/test.sh; else echo "no mayhem/test.sh"; fi

LABEL org.opencontainers.image.source="${REPO_URL}" \
      org.opencontainers.image.revision="${COMMIT}" \
      rlenv.tier="commit" \
      rlenv.fix="abc8f77c6a1e57c6ba12c77d47026e8602ff93f5" \
      rlenv.source_date_epoch="${SOURCE_DATE_EPOCH}" \
      rlenv.user="mayhem"

USER mayhem
WORKDIR /mayhem

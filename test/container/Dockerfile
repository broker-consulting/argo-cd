FROM docker.io/library/redis:7.2.4@sha256:7dd707032d90c6eaafd566f62a00f5b0116ae08fd7d6cbbb0f311b82b47171a2 as redis

# There are libraries we will want to copy from here in the final stage of the
# build, but the COPY directive does not have a way to determine system
# architecture, so we create a symlink here to facilitate copying.
RUN ln -s /usr/lib/$(uname -m)-linux-gnu /usr/lib/linux-gnu

# Please make sure to also check the contained yarn version and update the references below when upgrading this image's version
FROM docker.io/library/node:21.7.3@sha256:162d92c5f1467ad877bf6d8a098d9b04d7303879017a2f3644bfb1de1fc88ff0 as node

FROM docker.io/library/golang:1.22.1@sha256:0b55ab82ac2a54a6f8f85ec8b943b9e470c39e32c109b766bbc1b801f3fa8d3b as golang

FROM docker.io/library/registry:2.8@sha256:fb9c9aef62af3955f6014613456551c92e88a67dcf1fc51f5f91bcbd1832813f as registry

FROM docker.io/bitnami/kubectl:1.27@sha256:14ab746e857d96c105df4989cc2bf841292f2d143f7c60f9d7f549ae660eab43 as kubectl

FROM docker.io/library/ubuntu:22.04@sha256:1b8d8ff4777f36f19bfe73ee4df61e3a0b789caeff29caa019539ec7c9a57f95

ENV DEBIAN_FRONTEND=noninteractive
RUN  apt-get update && apt-get install --fix-missing -y \
    ca-certificates \
    curl \
    openssh-server \
    nginx \
    fcgiwrap \
    git \
    git-lfs \
    gpg \
    jq \
    make \
    wget \
    gcc \
    g++ \
    sudo \
    tini \
    zip && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

COPY --from=golang /usr/local/go /usr/local/go

COPY --from=kubectl /opt/bitnami/kubectl/bin/kubectl /usr/local/bin/kubectl

ENV PATH /dist:/go/bin:/usr/local/go/bin:/go/src/github.com/argoproj/argo-cd/dist:$PATH
ENV GOROOT /usr/local/go
ENV GOPATH /go

# Install build and test dependencies
COPY hack/install.sh hack/tool-versions.sh go.* ./
COPY hack/installers installers

RUN ./install.sh helm && \
    ./install.sh kustomize && \
    ./install.sh codegen-tools && \
    ./install.sh codegen-go-tools && \
    ./install.sh lint-tools && \
    go install github.com/mattn/goreman@latest && \
    go install github.com/kisielk/godepgraph@latest && \
    go install github.com/jstemmer/go-junit-report@latest && \
    rm -rf /tmp/dl && \
    rm -rf /tmp/helm && \
    rm -rf /tmp/ks_*

# These are required for running end-to-end tests
COPY ./test/fixture/testrepos/id_rsa.pub /root/.ssh/authorized_keys
COPY ./test/fixture/testrepos/nginx.conf /etc/nginx/nginx.conf
COPY ./test/fixture/testrepos/sudoers.conf /etc/sudoers
COPY ./test/fixture/testrepos/ssh_host_*_key* /etc/ssh/

# Copy redis binaries to the image
COPY --from=redis /usr/local/bin/* /usr/local/bin/

# Copy redis dependencies/shared libraries
# Ubuntu 22.04+ has moved to OpenSSL3 and no longer provides these libraries
COPY --from=redis /usr/lib/linux-gnu/libssl.so.3 /usr/lib/linux-gnu/
COPY --from=redis /usr/lib/linux-gnu/libcrypto.so.3 /usr/lib/linux-gnu/
RUN mv /usr/lib/linux-gnu/libssl.so.3 /usr/lib/$(uname -m)-linux-gnu/ && \
    mv /usr/lib/linux-gnu/libcrypto.so.3 /usr/lib/$(uname -m)-linux-gnu/ && \
    rm -rf /usr/lib/linux-gnu/

# Copy registry binaries to the image
COPY --from=registry /bin/registry /usr/local/bin/
COPY --from=registry /etc/docker/registry/config.yml /etc/docker/registry/config.yml

# Copy node binaries
COPY --from=node /usr/local/lib/node_modules /usr/local/lib/node_modules
COPY --from=node /usr/local/bin/node /usr/local/bin
COPY --from=node /opt/yarn-v1.22.19 /opt/yarn-v1.22.19

# Entrypoint is required for container's user management
COPY ./test/container/entrypoint.sh /usr/local/bin

ARG UID

# Prepare user configuration & build environments
RUN useradd -l -u ${UID} -d /home/user -s /bin/bash user && \
    echo "user ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/user && \
    mkdir -p /home/user/.kube && \
    mkdir -p /home/user/.cache && \
    chown -R user /home/user && \
    chgrp -R user /home/user && \
    HOME=/home/user git config --global user.name "ArgoCD Test User" && \
    HOME=/home/user git config --global user.email "noreply@example.com" && \
    mkdir -p /go/pkg && \
    mkdir -p /var/run/sshd && \
    mkdir -p /root/.ssh && \
    mkdir -p /go && \
    chown root /etc/ssh/ssh_host_*_key* && \
    chmod 0600 /etc/ssh/ssh_host_*_key && \
    mkdir -p /tmp/go-build-cache && \
    ln -s /usr/local/bin/node /usr/local/bin/nodejs && \
    ln -s /usr/local/lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm && \
    ln -s /usr/local/lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx && \
    ln -s /opt/yarn-v1.22.19/bin/yarn /usr/local/bin/yarn && \
    ln -s /opt/yarn-v1.22.19/bin/yarnpkg /usr/local/bin/yarnpkg && \
    mkdir -p /var/lib/registry

ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]

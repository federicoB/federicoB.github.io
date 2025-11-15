FROM debian:12

RUN apt-get update && apt-get install -y curl tar git && \
    curl -LO https://github.com/getzola/zola/releases/download/v0.21.0/zola-v0.21.0-x86_64-unknown-linux-gnu.tar.gz && \
    tar -xzf zola-v0.21.0-x86_64-unknown-linux-gnu.tar.gz && \
    mv zola /usr/local/bin/ && \
    rm zola-v0.21.0-x86_64-unknown-linux-gnu.tar.gz



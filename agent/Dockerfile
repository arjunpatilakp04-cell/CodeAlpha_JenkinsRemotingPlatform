FROM jenkins/inbound-agent:3384.v60d89463d9e0-1-jdk17

USER root

# Add common build tools ??? extend this list as your pipelines need more
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        git \
        curl \
        unzip \
    && rm -rf /var/lib/apt/lists/*

USER jenkins

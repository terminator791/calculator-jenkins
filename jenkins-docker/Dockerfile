FROM jenkins/jenkins:lts
USER root
# Install Docker CLI (client saja; daemon tetap pakai host via socket)
RUN apt-get update && apt-get install -y docker.io && rm -rf /var/lib/apt/lists/*
# Tambahkan user jenkins ke group docker agar bisa akses socket
RUN groupadd -f docker && usermod -aG docker jenkins
USER jenkins

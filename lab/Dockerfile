# Image "serveur cible" du lab : n'importe quelle image Debian + python3.
# Ansible est agentless : il lui faut seulement un accès (ici docker exec,
# en prod SSH) et un interpréteur Python sur la cible.
ARG BASE_IMAGE=debian:bookworm
FROM ${BASE_IMAGE}

RUN apt-get update \
 && apt-get install -y --no-install-recommends python3 sudo openssl procps ca-certificates \
 && rm -rf /var/lib/apt/lists/*

CMD ["sleep", "infinity"]

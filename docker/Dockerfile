FROM docker.io/estadrone2020/covapsy-jazzy:2026.1

# L'image de base tourne en tant que "ubuntu" : passer root pour installer les certificats
USER root
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=hostcerts /*.crt /usr/local/share/ca-certificates/
RUN update-ca-certificates

# Revenir à l'utilisateur par défaut de l'image
USER ubuntu
CMD [ "bash" ]

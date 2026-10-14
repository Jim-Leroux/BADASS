# Image routeur : FRR avec BGP, OSPF et IS-IS actifs, sans IP par défaut
FROM frrouting/frr:latest

COPY ./daemons.conf /etc/frr/daemons.conf
RUN chown frr:frr /etc/frr/daemons.conf

# GNS3 : laisser Start command vide (ou sh) — lance FRR puis ouvre un shell
CMD ["sh", "-c", "/usr/lib/frr/docker-start & exec sh"]

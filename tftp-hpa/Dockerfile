ARG ALPINE_VERSION=3.24
FROM alpine:${ALPINE_VERSION}

RUN apk upgrade --no-cache \
 && apk add --no-cache tftp-hpa \
 && mkdir -p /data

VOLUME ["/data"]
EXPOSE 69/udp

CMD ["in.tftpd", "-L", "-vvv", "-u", "root", "--secure", "--create", "/data"]

LABEL org.opencontainers.image.source="https://github.com/cedricfarinazzo/docker-images" \
      org.opencontainers.image.licenses="MIT" \
      org.opencontainers.image.title="tftp-hpa" \
      org.opencontainers.image.description="TFTP server (tftp-hpa) on alpine ${ALPINE_VERSION}" \
      io.cedricfarinazzo.alpine-version="${ALPINE_VERSION}"

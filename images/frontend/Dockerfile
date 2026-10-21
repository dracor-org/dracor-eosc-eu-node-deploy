# DraCor frontend 2.7.1, adapted to run under an arbitrary user ID in group 0
# (OpenShift/OKD restricted SCC, e.g. the EOSC EU Node Cloud Container Platform).
# Same site and nginx configuration as upstream, on the unprivileged nginx
# image (same nginx version), which listens on 8080 instead of 80.
FROM dracor/frontend:2.7.1@sha256:1f66a854e792149831a9c69d3f615063625394916e4b9c73a3172724dfe1cef5 AS upstream

FROM nginxinc/nginx-unprivileged:1.30.4-alpine@sha256:adf5042a17f4ecdd200c595fa9ffd1be37efb18f89a830bd1a00e4ab4d59d42c

COPY --from=upstream /usr/share/nginx/html /usr/share/nginx/html
COPY --from=upstream /etc/nginx/templates/default.conf.template /etc/nginx/templates/default.conf.template

USER root
RUN sed -i 's/listen\(\s*\)80;/listen\18080;/; s/listen\(\s*\)\[::\]:80;/listen\1[::]:8080;/' \
      /etc/nginx/templates/default.conf.template \
 && rm -f /etc/nginx/conf.d/default.conf
USER 101

EXPOSE 8080

LABEL org.opencontainers.image.title="DraCor frontend (OKD-ready)" \
      org.opencontainers.image.description="dracor/frontend:2.7.1 site and config on nginx-unprivileged, port 8080" \
      org.opencontainers.image.base.name="docker.io/dracor/frontend:2.7.1"

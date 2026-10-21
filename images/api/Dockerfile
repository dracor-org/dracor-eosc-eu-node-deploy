# DraCor API 1.4.1, adapted to run under an arbitrary user ID in group 0
# (OpenShift/OKD restricted SCC, e.g. the EOSC EU Node Cloud Container Platform).
# Changes: the paths eXist writes at startup become group-0-writable, and the
# placeholder secrets of the published upstream image are removed.
FROM dracor/api:1.4.1@sha256:cc027fc6097e6311711ab5f04d195db44031aaa92dfe7103e8ee56b2a758777d

# Empty secrets: the GitHub webhook is switched off (it answers 501); set a
# real value from a Secret only if you want to use the webhook. Both values
# are written into the database on the first start, so set them before that.
ENV GITHUB_WEBHOOK_SECRET="" \
    FUSEKI_SECRET=""

USER root
# etc/: the entrypoint rewrites the config files at every start
# data/, logs/: database and logs (mount volumes here in production)
# autodeploy/: upstream ships functx-1.0.1.xar readable by its owner only;
#   without this, the DraCor app fails to compile and the API answers 405
RUN chgrp -R 0 /opt/exist/etc /opt/exist/data /opt/exist/logs /opt/exist/autodeploy \
 && chmod -R g=u /opt/exist/etc /opt/exist/data /opt/exist/logs /opt/exist/autodeploy
USER 1001

LABEL org.opencontainers.image.title="DraCor API (OKD-ready)" \
      org.opencontainers.image.description="dracor/api:1.4.1 with group-0-writable paths for arbitrary user IDs, no placeholder secrets" \
      org.opencontainers.image.base.name="docker.io/dracor/api:1.4.1"

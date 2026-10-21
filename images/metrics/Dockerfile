# DraCor metrics 1.5.3, adapted to run under an arbitrary user ID in group 0
# (OpenShift/OKD restricted SCC, e.g. the EOSC EU Node Cloud Container Platform).
# The upstream image keeps its virtualenv under /root, which another user
# cannot read, so `poetry run` tries to create a new one at startup and fails.
# Here the dependencies are installed into the system Python at build time
# and the service is started directly.
FROM dracor/metrics:1.5.3@sha256:4fae7f1a33a59917a541a95ab2a21f181e498cc82d6530f91036a7abc92e5eeb

# Remove the old virtualenv first, otherwise poetry reuses it instead of
# installing into the system Python.
RUN rm -rf /root/.cache/pypoetry/virtualenvs \
 && POETRY_VIRTUALENVS_CREATE=false poetry install --without dev --no-interaction \
 && rm -rf /root/.cache
USER 1001

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8030"]

LABEL org.opencontainers.image.title="DraCor metrics (OKD-ready)" \
      org.opencontainers.image.description="dracor/metrics:1.5.3 with dependencies installed at build time, non-root" \
      org.opencontainers.image.base.name="docker.io/dracor/metrics:1.5.3"

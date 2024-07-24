# Stage 1: Build the watchdog
ARG PYTHON_VERSION=3.12
ARG DEBIAN_OS=slim-bookworm
FROM --platform=${TARGETPLATFORM:-linux/amd64} ghcr.io/openfaas/of-watchdog:0.10.4 AS watchdog

# Stage 2: Build the Python environment
FROM --platform=${TARGETPLATFORM:-linux/amd64} python:${PYTHON_VERSION}-${DEBIAN_OS} AS build

# Copy watchdog binary
COPY --from=watchdog /fwatchdog /usr/bin/fwatchdog
RUN chmod +x /usr/bin/fwatchdog

# Install additional packages
ARG ADDITIONAL_PACKAGE
RUN apt-get -qy update \
    && apt-get -qy install ${ADDITIONAL_PACKAGE} \
    && rm -rf /var/lib/apt/lists/*

# Add non-root user
RUN addgroup --system app && adduser app --system --ingroup app
RUN mkdir -p /home/app && chown app /home/app

# Switch to non-root user
USER app

# Set environment variables for user-specific installations
ENV PATH=$PATH:/home/app/.local/bin
ENV PYTHONUSERBASE=/home/app/.local

WORKDIR /home/app/

# Copy and install primary requirements
COPY --chown=app:app index.py           .
COPY --chown=app:app requirements.txt   .

# Install packages in the user-specific directory
RUN pip3 install --no-cache-dir --user -r requirements.txt

# Prepare function directory
RUN mkdir -p function
RUN touch ./function/__init__.py
WORKDIR /home/app/function/

# Copy and install function-specific requirements
COPY --chown=app:app function/requirements.txt	.

# Install packages in the user-specific directory
RUN pip3 install --no-cache-dir --user -r requirements.txt

# Copy function code
COPY --chown=app:app function/   .

# Stage 3: Test the build
FROM build AS test

ARG TEST_COMMAND=tox
ARG TEST_ENABLED=true
RUN [ "$TEST_ENABLED" = "false" ] && echo "skipping tests" || eval "$TEST_COMMAND"

# Stage 4: Finalize the image for shipping
FROM build AS ship
WORKDIR /home/app/

USER app

# Set up of-watchdog for HTTP mode
ENV fprocess="python index.py"
ENV cgi_headers="true"
ENV mode="http"
ENV upstream_url="http://127.0.0.1:5000"

HEALTHCHECK --interval=5s CMD [ -e /tmp/.lock ] || exit 1

CMD ["fwatchdog"]


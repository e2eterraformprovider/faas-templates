# Stage 1: Watchdog for OpenFaaS
ARG PYTHON_VERSION=3.11
FROM --platform=${TARGETPLATFORM:-linux/amd64} ghcr.io/openfaas/of-watchdog:0.9.15 as watchdog

# Stage 2: PyTorch Base Image with CUDA
# Replace NVIDIA CUDA base image with PyTorch CUDA base image
FROM --platform=${TARGETPLATFORM:-linux/amd64} pytorch/pytorch:2.0.1-cuda11.7-cudnn8-runtime as build

# Copy watchdog
COPY --from=watchdog /fwatchdog /usr/bin/fwatchdog
RUN chmod +x /usr/bin/fwatchdog

ARG ADDITIONAL_PACKAGE
ENV DEBIAN_FRONTEND=noninteractive

# Install Python and pip if needed (this is already in the PyTorch image, but you can specify additional packages)
RUN apt-get -qy update \
    && apt-get -qy install ${ADDITIONAL_PACKAGE} \
    && rm -rf /var/lib/apt/lists/*

# Add non-root user
RUN addgroup --system app && adduser app --system --ingroup app
RUN chown app /home/app
USER app

# Set PATH
ENV PATH=$PATH:/home/app/.local/bin

WORKDIR /home/app/

# Copy files and install dependencies
COPY --chown=app:app index.py           .
COPY --chown=app:app requirements.txt   .
USER root
RUN pip install --no-cache-dir -r requirements.txt
USER app

# Create the function folder
RUN mkdir -p function
RUN touch ./function/__init__.py
# Install additional requirements for the function
WORKDIR /home/app/function/
COPY --chown=app:app function/requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Copy function code
USER root
COPY --chown=app:app function/   .

# Stage 3: Test (optional)
FROM build as test
ARG TEST_COMMAND=tox
ARG TEST_ENABLED=true
RUN [ "$TEST_ENABLED" = "false" ] && echo "skipping tests" || eval "$TEST_COMMAND"

# Stage 4: Ship
FROM build as ship
WORKDIR /home/app/
USER app

# Set up of-watchdog for HTTP mode
ENV fprocess="python3 index.py"
ENV cgi_headers="true"
ENV mode="http"
ENV upstream_url="http://127.0.0.1:5000"

HEALTHCHECK --interval=5s CMD [ -e /tmp/.lock ] || exit 1

# Start the watchdog
CMD ["fwatchdog"]

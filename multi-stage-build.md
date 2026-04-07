Option 2: Manual FIPS Pairing in Multi-Stage Build

Dockerfile with helper stage:

⁠ dockerfile
# Stage 1: Build application
FROM ubuntu:22.04 AS builder
RUN apt-get update && apt-get install -y python3 pip openssl
COPY . /app
RUN pip install fastanpr

# Stage 2: Helper container - generate correct FIPS config
FROM gcr.io/distroless/base-debian12 AS fips-helper
# Temporarily get openssl in distroless
COPY --from=alpine:latest /usr/bin/openssl /tmp/openssl
COPY --from=alpine:latest /usr/lib/ssl /tmp/ssl
ENV LD_LIBRARY_PATH=/tmp
# Generate paired config
RUN /tmp/openssl fipsinstall -module /path/to/fips.so -out /tmp/fipsmodule.cnf

# Stage 3: Final distroless image
FROM gcr.io/distroless/base-debian12
# Copy paired FIPS files from helper
COPY --from=fips-helper /tmp/fipsmodule.cnf /etc/ssl/fipsmodule.cnf
COPY --from=builder /path/to/fips.so /usr/lib/ssl/modules/fips.so
COPY --from=builder /app /app
# Critical: Tell OpenSSL where to find the config
ENV OPENSSL_CONF=/etc/ssl/fipsmodule.cnf
ENV OPENSSL_MODULES=/usr/lib/ssl/modules
CMD ["python", "/app/your_script.py"]
 ⁠

Build command:

⁠ bash
docker build --target fips-helper -t fips-temp:latest .
docker build -t your-app:latest .

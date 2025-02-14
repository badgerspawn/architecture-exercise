# CICD notes

## docker files

Use distroless base images for small image size and minimal attack vector

Main app:

```sh
# Use a secure, minimal Node.js image
FROM gcr.io/distroless/nodejs18-debian12:<specific digest version>

# Set non-root user explicitly
USER nonroot

# Set the working directory
WORKDIR /app

# Copy the tar file and extract it
ARG TAR_FILE=app.tar
COPY ${TAR_FILE} /app/app.tar
RUN tar -xf app.tar --strip-components=1 && rm app.tar

# Expose only necessary ports (my solution is expecting https)
EXPOSE 443

# Set node environment for security
ENV NODE_ENV=production

# Run with a read-only filesystem
CMD ["server.js"]
```

Authenticator app:

```sh
FROM gcr.io/distroless/java17:nonroot

# Set the working directory
WORKDIR /app

# Copy the JAR file with verification (supplied via CICD or KV fteched secret)
ARG JAR_FILE=app.jar
COPY target/${JAR_FILE} app.jar
RUN echo "<expected-sha256> app.jar" | sha256sum --check

# Expose only necessary ports
EXPOSE 443

# Run application as non-root user
USER nonroot:nonroot

# Run the application with a read-only filesystem
CMD ["java", "-jar", "/app/app.jar"]
```

Cleanup Job

```sh
FROM gcr.io/distroless/python3:<specific digest version>

# Set non-root user explicitly
USER nonroot

# Set the working directory
WORKDIR /app

# Copy the script into the container
ARG CLEANUP_SCRIPT=script.py
COPY ${CLEANUP_SCRIPT} script.py

# Copy dependency file and install securely
COPY requirements.txt requirements.txt
RUN python -m venv /app/venv && \
    /app/venv/bin/pip install --no-cache-dir -r requirements.txt && \
    rm -rf ~/.cache/pip

# Expose necessary ports (if applicable)
# EXPOSE 8080  # Uncomment if needed

# Set environment variables for security
ENV PYTHONUNBUFFERED=1 \
    PATH="/app/venv/bin:$PATH"

# Run with a read-only files
CMD ["script.py"]
```

## Deployment actions

Example github action to build and tag to ACR:

```yaml
name: Docker Build and Push to ACR

on:
  push:
    tags:
      - 'v*'

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Log in to Azure Container Registry
        uses: azure/docker-login@v1
        with:
          login-server: ${{ secrets.ACR_LOGIN_SERVER }}
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}

      - name: Extract tag name
        id: extract_tag
        run: echo "::set-output name=TAG::${GITHUB_REF#refs/tags/}"

      - name: Build Docker image
        run: |
          docker build -t ${{ secrets.ACR_LOGIN_SERVER }}/my-app:${{ steps.extract_tag.outputs.TAG }} .
          docker tag ${{ secrets.ACR_LOGIN_SERVER }}/my-app:${{ steps.extract_tag.outputs.TAG }} ${{ secrets.ACR_LOGIN_SERVER }}/my-app:latest

      - name: Push Docker image
        run: |
          docker push ${{ secrets.ACR_LOGIN_SERVER }}/my-app:${{ steps.extract_tag.outputs.TAG }}
          docker push ${{ secrets.ACR_LOGIN_SERVER }}/my-app:latest
```

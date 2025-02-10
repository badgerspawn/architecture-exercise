# CICD notes

## docker files

Use distroless base images for small image size and minimal attack vector

Main app:

```sh
FROM gcr.io/distroless/java17

# Set the working directory
WORKDIR /app

# Copy the JAR file from the build artifact location to the container
ARG JAR_FILE=app.jar
COPY target/${JAR_FILE} app.jar

# Expose the port (adjust based on your app settings)
EXPOSE 443

# Run the application with a non-root user (default in distroless)
CMD ["java", "-jar", "app.jar"]
```

Authenticator app:

```sh
FROM gcr.io/distroless/java17

# Set the working directory
WORKDIR /app

# Copy the JAR file from the build artifact location to the container
ARG JAR_FILE=app.jar
COPY target/${JAR_FILE} app.jar

# Expose the port (adjust based on your app settings)
EXPOSE 8080

# Run the application with a non-root user (default in distroless)
CMD ["java", "-jar", "app.jar"]
```

Cleanup Job

```sh
FROM python:3.9-slim

# Set the working directory to /app
WORKDIR /app

# Copy the Python script from the build artifact location to the container
ARG CLEANUP_SCRIPT=script.py
COPY target/${CLEANUP_SCRIPT} script.py

# Install any dependencies required by the script
RUN pip install -r requirements.txt

# Make the script executable
RUN chmod +x script.py

# Set the command to run when the container starts
CMD ["python", "script.py"]
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
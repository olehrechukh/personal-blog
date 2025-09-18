---
title: Local Azure Storage Development with Azurite and Docker
description: Create a reproducible local Azure Storage environment using Azurite and Docker. Includes Docker Compose and docker run examples, an automated initialization script, and guidance for integrating Azurite with your application tests.
date: 2025-09-18
tags: [Azure, Azurite, Docker,  DevOps, Testing]
---

# Local Azure Storage Development Made Easy with Azurite and Docker

*A practical guide to spinning up a local, high-fidelity Azure Storage emulator for cleaner testing and faster development cycles.*

<figure>
  {% image "./data/azurit.jpg", "Sample workbook visualizations showing combined charts and metrics used in the demo workbook" %}
</figure> 

Testing an application that reads and writes from Azure Storage can be a hassle. Do you mock the storage SDKs? Do you point your development machine to a real (and costly) Azure Storage account? Both have significant downsides, from inaccurate mocks to network latency and cost management.

There’s a better way: **Azurite**, the official open-source emulator for Azure Storage.

This guide will show you how to use Docker to create a reliable, local, and disposable Azurite environment, complete with an automated setup script. We'll cover two methods: a simple approach with Docker Compose and a look under the hood with `docker run` commands.

### What is Azurite?

Azurite provides a free local environment for testing your Azure Blob, Queue, and Table storage applications. It emulates the Azure Storage APIs with high fidelity, meaning the code you write against Azurite will behave the same way it does against a real Azure Storage account. By running it in a Docker container, we get a clean, isolated, and easily reproducible setup.

### Automating Setup with the Azure CLI

While you can connect to Azurite and create containers manually, a better approach for a clean, repeatable development environment is to automate this setup.

In this guide, we will use a separate Docker container running the official **Azure CLI** (`mcr.microsoft.com/azure-cli`) to automatically create a blob container every time our environment starts. This "task runner" container will execute a simple shell script, ensuring our Azurite instance is always in the state our application expects, with no manual steps required.

### Method 1: The Simple Life with Docker Compose

For multi-container applications, Docker Compose is the gold standard. It lets us define and run our entire local environment with a single file and command.

Our setup consists of two services:
1.  `azurite`: The storage emulator itself.
2.  `container-creator`: A one-off utility container that runs a script to initialize our storage environment (e.g., create a default container).

Here’s the `docker-compose.yml`:

```yaml
services:
  azurite:
    image: mcr.microsoft.com/azure-storage/azurite
    restart: always
    command: >
      azurite --loose
      --blobHost 0.0.0.0 --blobPort 10000
      --queueHost 0.0.0.0 --queuePort 10001
      --tableHost 0.0.0.0 --tablePort 10002
      --location /workspace --debug /workspace/debug.log
    ports:
      - "10000:10000"
      - "10001:10001"
      - "10002:10002"

  container-creator:
    image: mcr.microsoft.com/azure-cli:latest
    depends_on:
      - azurite
    environment:
      CONTAINER_NAME: images
      AZURE_STORAGE_CONNECTION_STRING: DefaultEndpointsProtocol=http;AccountName=devstoreaccount1;AccountKey=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==;BlobEndpoint=http://azurite:10000/devstoreaccount1;
    volumes:
      - ./blob-storage:/blob-storage
    entrypoint: ["/bin/sh", "-c", "chmod +x /blob-storage/entrypoint.sh && /blob-storage/entrypoint.sh"]
```

**To run it, simply execute:**

```bash
# Use --build if you change the entrypoint.sh script
docker-compose up --build -d
```

When you're done, tear it all down with:

```bash
docker-compose down
```

This is the cleanest and most recommended approach.

You can find the complete docker-compose.yml and initialization scripts in this [GitHub repository](https://github.com/olehrechukh/azurite-container)

### Method 2: Under the Hood with `docker run`

Understanding what Docker Compose does behind the scenes is valuable. Here’s how to achieve the same setup using `docker run` commands.

The key is creating a shared network so the `container-creator` container can communicate with the `azurite` container using its name as a hostname.

**Step 1: Create the Network**
```bash
docker network create azurite-network
```

**Step 2: Start the Azurite Container**
This command starts the Azurite service and attaches it to our network.
```bash
docker run -d --name azurite --restart always \
  --network azurite-network \
  -p 10000:10000 -p 10001:10001 -p 10002:10002 \
  mcr.microsoft.com/azure-storage/azurite \
  azurite --loose --blobHost 0.0.0.0 --blobPort 10000 --queueHost 0.0.0.0 --queuePort 10001 --tableHost 0.0.0.0 --tablePort 10002 --location /workspace --debug /workspace/debug.log
```

**Step 3: Run the Initialization Script**
This command starts the `container-creator` container using the `mcr.microsoft.com/azure-cli` image. The container runs the setup script and then exits. Notice how the connection string points to `http://azurite:10000`.
```bash
docker run --rm --network azurite-network \
  -e CONTAINER_NAME=images \
  -e AZURE_STORAGE_CONNECTION_STRING="..." \
  -v $(pwd)/blob-storage:/blob-storage \
  mcr.microsoft.com/azure-cli:latest \
  /bin/sh -c "chmod +x /blob-storage/entrypoint.sh && /blob-storage/entrypoint.sh"
```

### Method 3: Using a Custom Application as a Task Runner

For more complex initialization logic, you aren't limited to using shell scripts and the Azure CLI. A more powerful and flexible approach is to write a small, custom application to act as your setup task runner.

The idea is simple:
1.  **Write a small application** in a language like Python, Node.js, or C# that uses the official Azure Storage SDK. This application would contain the logic to connect to Azurite and set up your desired state (e.g., create multiple containers, upload default data, set metadata).
2.  **Package this application** into its own Docker image.
3.  **Replace the `container-creator` service** in the `docker-compose.yml` file to use your new custom image instead of the `azure-cli` image.

This method gives you the full power of a programming language to perform sophisticated setup tasks, making it ideal for complex scenarios while keeping your environment fully automated.

### The Auto-Setup Script

The magic of the `container-creator` is in its entrypoint script. It uses the Azure CLI to prepare our storage environment, so our application doesn't have to worry about creating its own containers on startup. This latest version of the script also verifies that the container was created successfully.

Here is the `blob-storage/entrypoint.sh` script:
```bash
#!/bin/sh
set -eu

CONTAINER_NAME="${CONTAINER_NAME:?Error: CONTAINER_NAME is required but not set}"
AZURE_STORAGE_CONNECTION_STRING="${AZURE_STORAGE_CONNECTION_STRING:?Error: AZURE_STORAGE_CONNECTION_STRING is required but not set}"

echo "Creating container: $CONTAINER_NAME ..."
az storage container create \
  --name "$CONTAINER_NAME" \
  --public-access blob \
  --connection-string "$AZURE_STORAGE_CONNECTION_STRING"

echo "Verifying container '$CONTAINER_NAME'..."

container_info=$(az storage container show \
  --name "$CONTAINER_NAME" \
  --connection-string "$AZURE_STORAGE_CONNECTION_STRING" 2>/dev/null || true)

if [ -z "$container_info" ]; then
  echo "❌ Error: Container '$CONTAINER_NAME' was not found after creation." >&2
  exit 1
else
  echo "✅ Container '$CONTAINER_NAME' exists and is accessible."
fi

echo "Done."
```

### Connecting Your Application

Whether you used Docker Compose or `docker run`, your local Azurite instance is now running and accessible. To connect your application to it, use the standard Azurite connection string:

```
DefaultEndpointsProtocol=http;AccountName=devstoreaccount1;AccountKey=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==;BlobEndpoint=http://127.0.0.1:10000/devstoreaccount1;
```

### Conclusion

By combining Azurite and Docker, you can create a powerful, isolated, and automated local development environment for Azure Storage. This approach eliminates flaky mocks and the costs of live cloud resources, allowing you to build and test with confidence.

Happy coding!

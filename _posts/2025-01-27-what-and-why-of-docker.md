---
title: "What and Why of Docker"
date: 2025-11-01 12:00:00 +0000
categories: [Docker]
tags: [docker, R1]
---


Before programs were deployed on docker, we had VMs and before that we had bare metals. The problem with bare metals is that it's very hard to come up with a specs that fits the requirement correctly. The number of users might go up with time. The traffic can increase. Most of the people ended up with machines that had more resources than they used.

After this we had VMs. VMs allows users to run multiple OSs in a single machine through hypervisor. All OSs share resources of the system. When using a hypervisor, say vmware, when you start an OS inside it you could have seen an option to configure RAM, CPU and storage. Cloud providers buy big machines and provide VMs to end users as a service.

The problem with VMs is that it provides isolation at a high cost. You run multiple OSs in the same system and running an OS requires resources. Enter docker. Docker runs containers on top of the host OS by sharing kernel. Container is an instance of image. Image is sort of a recipe on how how to build a container instance. The image is defined in Dockerfile. It looks like the following:
```
FROM python:3.13
WORKDIR /usr/local/app

# Install the application dependencies
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

# Copy in the source code
COPY src ./src
EXPOSE 8080

# Setup an app user so the container doesn't run as the root user
RUN useradd app
USER app

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

Docker also solves the problem of 'it works on my machine'. It's super easy to replicate the environment across different machines.

When `docker run -itd ...` is executed, the following things happen behind the scenes:
1. Send http request to docker daemon to run an instance of image.
2. dockerd makes grpc call to containerd.
3. containerd tries to find the image locally, if not it tries to pull it from a remote server, maybe dockerhub.
4. containerd calls runc to run the container.
5. runc is the one who sets PID, network, mount, user, namespace and cgroups.

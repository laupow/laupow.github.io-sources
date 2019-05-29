---
title: "GKE v1.13 Upgrade"
date: 2019-05-28
draft: false
tags: ["kubernetes", "docker", "gke"]
---
We recently upgraded a Kubernetes cluster to GKE v1.13.5. 

After the upgrade, the `docker` client was no longer responsive and crashed hard. 
```bash
$ docker ps
Segmentation fault (core dumped)
```

Uh-oh.

This cluster runs a Jenkins instance which launches build containers in the cluster.
Each build container eventually runs Docker commands like 
`docker build -t ${env.IMAGE_NAME} -f path/to/Dockerfile .` 
to build container images. 
We mount the Docker client into the build container from the [Container-Optimized OS](https://cloud.google.com/container-optimized-os/) host VM. 
 
A simplified Pod definition of the build container:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: build-container
spec:
  containers:
    - name: build-container
      image: 'centos:7'
      command: 
        - sh
        - -c 
        - '<build command>'
      volumeMounts:
        - mountPath: /var/run/docker.sock
          name: docker-sock
        - mountPath: /usr/bin/docker
          name: docker
  volumes:
    - name: docker
      hostPath:
        path: /usr/bin/docker
        type: File
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
        type: Socket
```

Whether or not this should have ever worked is a fair question.
However, it worked for the past two years, and using the mounted `docker` client broke following an upgrade to GKE v1.13.
 
Our current workaround to the Docker client segmentation fault 
is to use the official [static Docker binaries](https://download.docker.com/linux/static/stable/x86_64/) in our build container.

There was never an issue reading or writing to the socket, and using official Docker client binaries seems to work. 

---
layout: post
title: ".NET Core on Google Kubernetes Engine"
date: 2018-07-18
categories: programming devops
---
High level steps:
Create .NET Core Web API app
Create Google Cloud Project
Dockerize .NET Core app
Create k8s cluster
Build Docker image
Push Docker image to repo
Create k8s "pod" using the image created above (AKA container)
- Cluster -> pod -> container (big to little)
Expose deployed container to external traffic
You made a internet - show your friends and loved ones




What's Docker? Docker is a container platform. There are other options for deploying containers, and the concept did not originate with them (as many crusty devops veterans will be eager to point out), but Docker is arguably the most popular and accessible.

What's an image? An image is like a snapshot of an application *and everything needed to run it*. Images are used to build containers...

What's a container? A container is a running instance of an image. It's a self-contained environment in which your application lives. If you're accustomed to deploying applications to a VM, containers abstract away much of the work of building and provisioning the machine that will be running your application. In other words, VMs utilize hardware virtualization - many instances of an OS running on a single server; containers utilize OS virtualization - many self-contained apps running in isolation on a single OS kernel.

The heck is a "Kubernetes"? Kubernetes is an open-source platform for hosting containers, originally created by Google.

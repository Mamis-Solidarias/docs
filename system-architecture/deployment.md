# Deployment

This system will be deployed in Azure, but there are several options we can evaluate. Out of all of them, we settled on 2 main options:

- **Kubernetes**
- **Azure Container Apps**

## Kubernetes

Kubernetes is a container orchestration service to deliver micro services.

Concepts:

- **Image**: our service

- **Pods**: Each image is deployed in a pod. It is basically as a container for the image. 
- **Deployment**: A deployment is a yaml file that indicate how many replicas (pods) have to be spawned with an image.
- **Service**: It is a load balancer that can handle traffic either from the outside world or from other pods, and it redirects it to one of the pods that deployment.

Kubernetes can automatically scale up or down this replicas as much as it needs to maintain performance.

### Tye



## Azure Container Apps




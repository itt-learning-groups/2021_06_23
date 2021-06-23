# Kubernetes "Lab" Exercise: Explore the EKS cluster + service a bit more

## June 23, 2021

## Prerequisites

* Set up an EKS cluster on AWS with a simple deployed `deployment` and `service` as described [here](https://github.com/us-learn-and-devops/2021_05_26/blob/main/README.md) in last week's group exercise.
* Open up the `webapp-svc.yaml` and `webapp-deployment.yaml` files that you used to deploy the service and deployment objects. We'll want to reference these.

## Lab

### Explore how a service object uses endpoints

* Use kubectl to match service endpoints with the IPs of pods that "belong" to the service:

  * Get the name of your nginx service: `kubectl get svc -n default | grep nginx`
  * Use that name to get the list of the service's endpoints: `kubectl describe svc nginx-svc -n default | grep Endpoints`
  * Find running pods that belong to the service by filtering by the service's selector label(s) (i.e. `env: dev` according to `webapp-svc.yaml`). Include pod IP by using the `-o wide` option, and compare the IPs to the service endpoints: `kubectl get pods -l env=dev -o wide -n default`.
  * Observe the match between service pod IPs and service endpoints if the pods change.
    * Kill one of the current service pods, e.g. `kubectl delete pod nginx-deployment-75ffc6d4d-kw79t -n default`.
    * Wait a moment or two, then again compare service pod IPs and service endpoints.
  * Recap: Identify and describe the two ways that a service object is associated with the pods that "belong" to it.

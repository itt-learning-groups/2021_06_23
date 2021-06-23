# Kubernetes "Lab" Exercise: Explore the EKS cluster + service a bit more

## June 23, 2021

## Prerequisites

* Set up an EKS cluster on AWS with a simple deployed `deployment` and `service` as described [here](https://github.com/us-learn-and-devops/2021_05_26/blob/main/README.md) in last week's group exercise.
* Open up the `webapp-svc.yaml` and `webapp-deployment.yaml` files that you used to deploy the service and deployment objects. We'll want to reference these.

## kubectl short resource-names

Here's a handy list of k8s resource names along with the short-form names you can use in a kubectl command (if the short-form name exists). If the short-form name exists, you can substitute it for the full resource-name. For example `kubectl get ev` works identically to `kubectl get events`.

* certificatesigningrequests (aka 'csr')
* clusters (valid only for federation apiservers)
* clusterrolebindings
* clusterroles
* componentstatuses (aka 'cs')
* configmaps (aka 'cm')
* daemonsets (aka 'ds')
* deployments (aka 'deploy')
* endpoints (aka 'ep')
* events (aka 'ev')
* horizontalpodautoscalers (aka 'hpa')
* ingresses (aka 'ing')
* jobs
* limitranges (aka 'limits')
* namespaces (aka 'ns')
* networkpolicies
* nodes (aka 'no')
* persistentvolumeclaims (aka 'pvc')
* persistentvolumes (aka 'pv')
* pods (aka 'po')
* poddisruptionbudgets (aka 'pdb')
* podsecuritypolicies (aka 'psp')
* podtemplates
* replicasets (aka 'rs')
* replicationcontrollers (aka 'rc')
* resourcequotas (aka 'quota')
* rolebindings
* roles
* secrets
* serviceaccounts (aka 'sa')
* services (aka 'svc')
* statefulsets
* storageclasses
* thirdpartyresources

## Lab

### Explore service objects

* Use kubectl to see how a service object uses endpoints:

  * Get the name of your nginx service: `kubectl get svc -n default | grep nginx`
  * Use that name to get the list of the service's endpoints: `kubectl describe svc nginx-svc -n default | grep Endpoints`
  * Find running pods that belong to the service by filtering by the service's selector label(s) (i.e. `env: dev` according to `webapp-svc.yaml`). Include pod IP by using the `-o wide` option, and compare the IPs to the service endpoints: `kubectl get pods -l env=dev -o wide -n default`.
  * Observe the match between service pod IPs and service endpoints if the pods change.
    * Kill one of the current service pods, e.g. `kubectl delete pod nginx-deployment-75ffc6d4d-kw79t -n default`.
    * Wait a moment or two, then again compare service pod IPs and service endpoints.
  * Recap: Identify and describe the two ways that a service object is associated with the pods that "belong" to it.

* Use kubectl to understand the coupling between pods and services

  * Question: What will happen to the service pods if you delete the service? Make a prediction.
  * Delete the service (only): `kubectl delete svc nginx-svc`
  * Attempt to re-list the service pods. What has changed? Was your prediction right?
  * Question: What will the service endpoints look like if you redeploy the service now? Make a prediction.
  * Redeploy the service: `kubectl apply -f ./webapp-svc.yaml`
  * Check the service pod IPs and service endpoints again, looking for a match.
  * What do these experiments tell you about the coupling between a service and the pods that "belong" to it?

### Explore deployment objects

* Use kubectl to understand the coupling between pods and deployments

  * Question: What will happen to the service pods if you delete the nginx deployment object? Make a prediction.
  * Delete the deployment (only): `kubectl delete deploy nginx-deployment -n default`
  * Attempt to re-list the pods. What has changed? What does this tell you about the way pods "belong" to a deployment vs the way they "belong" to a service?

* *A parenthetical final look at service endpoints*

  * *Question: What will the service endpoints look like if you redeploy the deployment object now? Make a prediction.*
  * *Redeploy the deployment: `kubectl apply -f ./webapp-deployment.yaml`*
  * *Wait a moment or two, then yet again compare the IPs to the service endpoints.*

* Create a 2nd deployment

  * Create a new file called `webapp-deployment-prod.yaml` and copy the contents of `webapp-deployment.yaml` into it. Change the deployment name to `nginx-deployment-prod` and change the labels on lines 6, 11, and 15 to `prod`. Also change the replica count on line 8 to `1`. (The latter will help us avoid a resource-limit shortage by keeping us at or below 4 total deployed pods.)
  * Deploy this alongside the `env: dev` deployment we already have up and running: `kubectl apply -f ./webapp-deployment-prod.yaml`.
  * Wait a moment, then check the pods list: `kubectl get pods -n default -o wide`
  * Filter for just the `prod` pods: `kubectl get pods -n default -l env=prod -o wide`
    * *Question: Does the `prod` pod "belong" to any service object right now? How could you check/verify? What does that mean for the prod pod right now? Could we add a service for it? How would we do that?*
  * *Housekeeping: If you didn't already delete it, clean up the `dev` deployment so we have fewer pods running in our very small-capacity cluster before the next step: `kubectl delete deploy nginx-deployment -n default`*

* Check out `deployment` object capabilities

  * `Deployment` objects can "scale".
    * Scale the `prod` deployment to 2: `kubectl scale deploy nginx-deployment-prod --replicas 2 -n default`
    * Wait a moment, then check the `prod` pods list again: `kubectl get pods -n default -l env=prod -o wide`
    * Note a few relevant lines from the deployment description: `kubectl describe deploy nginx-deployment-prod -n default | grep -E '^Replicas|^OldReplicaSets|^NewReplicaSet'`
  * Ability to scale is one of the important capabilities of a `deployment` object. Note that...
    * That ability is actually provided by the `replicaSet` object that a `deployment` object contains: It's not actually an ability of the `deployment` object itself. (You can create a replicaSet object on its own and use it to scale pods without creating a deployment object.)
    * That ability is manual only -- unless we employ more machinery. A replicaSet, on its own, can't autoscale. In some ways, a replicaSet looks a lot like the AWS auto-scaling groups we examined this spring; but they can't autoscale on their own. Remember this fact: We'll come back to it when we talk about pod autoscaling.
  * Another important capability of a `deployment` object is update management...

  * `Deployment` objects can do smooth version updates.
    * Note another relevant line from the deployment description: `kubectl describe deploy nginx-deployment-prod -n default | grep -E '^StrategyType'`
      * This should show that we're using the default strategy "Rolling Updates" for automagic zero-downtime updates.
    * Let's watch a rolling update by deploying a new version of the nginx-deployment-prod deployment with updated container `image: nginx:1.21.0-alpine`. (There's no significance to this particular image tag. The only significance is that it's different from `image: nginx:latest` so it constitutes a deployment version update if we apply it.)
      * Update the container image in your `webapp-deployment-prod.yaml` file and deploy the update via `kubectl apply -f ./webapp-deployment-prod.yaml` again
    * Use kubectl to describe the deployments again: `kubectl describe deploy nginx-deployment-prod -n default`
      * You may not be quick enough to "catch" the rolling update in-progress. But you can still note the replicaSet events listed in the event log at the bottom of the description output. And it you compare the `NewReplicaSet` to what it was a moment ago, you should see that it has changed.

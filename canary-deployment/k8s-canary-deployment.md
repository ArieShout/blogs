# Canary Deployment with Kubernetes

Canary deployment is a pattern that rolls out releases to a subset of users or servers. It deploys the changes to a
small set of servers, which allows you to test and monitor how the new release works before rolling the changes to the
rest of the servers.

Kubernetes is the popular Docker container orchestrator in the community. It has clean and separated definition for the
deployments which represents the project runtime and the services which represents the endpoints the users will access.
It connects the two types of resources using the labels and selectors. A deployment may be accessed from zero to many
service endpoints, and at the same time a service may route the traffic to zero to many deployments (just like a load balancer).

With this label-selector routing mechanism, it's easy to do canary deployment with Kubernetes:

1. Deploy the initial version of your application to Kubernetes (say, `deployment-A`), and deploy the service endpoint 
   that matches the label of `deployment-A`.

   This represents the initial state before we do the canary deployment.

1. Deploy the new version of your application to Kubernetes (say, `deployment-B`). It has the same label as required by
   the service endpoint (partially same with `deployment-A`).

1. Decrease the number of replicas in `deployment-A`, so that the replicas ratio of the initial release and the new 
   release matches the target.

   Now we are in a intermediate state where both new version and old version are serving the traffic.

1. When the new version of your application is validated to be working properly, we can make the full deployment by:

   1. Update the number of replicas in `deployment-B` to the target number
   1. Delete `deployment-A`

## Nginx Canary Deployment Example

We will demonstrate the canary deployment with Kubernetes using the public Nginx Docker image.

### Prepare the Initial Version

The initial state consists of a deployment of a old version of Nginx, and a service endpoint. This represents the initial
state of world we have before we start the canary deployment for the new version.

```sh
cat <<EOF >initial.yml
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment-1.12
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
        deployment: canary
        version: "1.12"
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.12
        ports:
        - containerPort: 80
---
kind: Service
apiVersion: v1
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
    deployment: canary
  ports:
    - port: 80
      targetPort: 80
EOF
kubectl apply -f initial.yml
kubectl get services nginx-service --watch
# wait until the external-ip is provisioned for the service
```

Note that:

* The `selector` in the service spec matches the `labels` in the deployment spec.
* The `replicas` is set to `2` for the deployment, which is the target replicas count for our application.

   This means we may only have one replica for the old version, and one replica for the new version (1:1) if we want to
   preserve the target replicas count. In real world application, the `replicas` may be larger which offers more choices
   of replica ratio.

### Deploy the New Version

If we want to deploy a new version of Nginx server in canary deployment pattern, we can create another deployment where

* The deployment name is different from the initial deployment
* The image version is `nginx:1.13`
* The `labels` are matched by the `nginx-service` in the initial state. It has different value for label `version` compared
   to the initial deployment
* The `replicas` is set to 1, which is the target replicas count for the new version

```sh
cat <<EOF >deployment-1.13.yml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment-1.13
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
        deployment: canary
        version: "1.13"
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.13
        ports:
        - containerPort: 80
EOF
kubectl apply -f deployment-1.13.yml
```

When the new deployment (`nginx-deployment-1.13`) is ready, some users should be able to see the web server served by
Nginx 1.13:

```sh
$ service_ip=$(kubectl get services nginx-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
$ curl -is "$service_ip" | grep Server:
Server: nginx/1.12.2
$ curl -is "$service_ip" | grep Server:
Server: nginx/1.13.9
$ curl -is "$service_ip" | grep Server:
Server: nginx/1.12.2
$ curl -is "$service_ip" | grep Server:
Server: nginx/1.13.9
$ curl -is "$service_ip" | grep Server:
Server: nginx/1.12.2
```

To be more careful and conservative, we may first create the deployment with labels that cannot be matched by the
service endpoint, for example, set `deployment: canary-prepare` first, and then check if it is working properly:

```sh
pod=$(kubectl get pods -l 'version=1.13' -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward "$pod" 8080:80
# now you can visit http://localhost:8080 to see if it is working properly
```

If the pods for the deployment is working, we can update the label of the deployment so that it can be matched by
the service endpoint.

### Update the Replicas Count

Now we have 3 working replicas, 2 on old version and 1 on new, and all of them may be accessed from the frontend
service `nginx-service`. If this is the desired state of the deployment, you may stop here.

If not, for example, we want to preserve the replicas count in the initial state, we can decrease the `replicas`
for the initial deployment `nginx-deployment-1.12`.

```sh
cat <<EOF >update-1.12.yml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment-1.12
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
        deployment: canary
        version: "1.12"
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.12
        ports:
        - containerPort: 80
EOF
kubectl apply -f update-1.12.yml
```

### Full Deployment

When the new version is verified to be working, we can perform the full deployment. We just need to increase the
`replicas` in `nginx-deployment-1.13` to `2`, and then delete deployment `nginx-deployment-1.12`.

```sh
cat <<EOF >update-1.13.yml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment-1.13
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
        deployment: canary
        version: "1.13"
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.13
        ports:
        - containerPort: 80
EOF
kubectl apply -f update-1.13.yml
# verify that all pods are up and working
kubectl delete deployment nginx-deployment-1.12
```

Now the users should only see the new version of your application.
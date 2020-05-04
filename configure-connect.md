## Configure Connect For Our Test App
We want to use Consul Connect for our test application.


**📝 Exercise: What's our next step?**

<details>
<summary>Hint</summary>

As it says in the docs: https://www.consul.io/docs/platform/k8s/connect.html we
need to add an annotation to our apps that tells Consul to inject the sidecar:

```yaml
"consul.hashicorp.com/connect-inject": "true"
```

---

</details>

👇

👇

👇

👇

👇

👇

👇

👇


## Redeploy DB

Edit `db-deployment.yaml` and add the annotation:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
  labels:
    app: db
spec:
  replicas: 2
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
      annotations:
        "consul.hashicorp.com/connect-inject": "true"
    spec:
      containers:
        - name: my-first-container-name
          image: nicholasjackson/fake-service:v0.7.8
          ports:
            - containerPort: 9090
          env:
          - name: MESSAGE
            value: "db"
```

**NOTE:** The annotation must be added to the `metadata` section of the template,
not the metadata of the Deployment. This is because when we perform the pod mutation
we only get access to the pod spec, not the whole deployment, so we won't see any
annotations on the deployment.

Redeploy the app:

```shell script
kubectl apply -f db-deployment.yaml
deployment.apps/db configured
```

Now run:

```shell script
kubectl get pod -l app=db
NAME                  READY   STATUS    RESTARTS   AGE
db-5f7ff65f54-jqn86   3/3     Running   0          4m3s
db-5f7ff65f54-zbmbm   3/3     Running   0          7m16s
```

You should see that the pods have `3/3`. This means they're actually running 3
containers. We've successfully injected the envoy sidecar and another helper container
we use to re-register the service if the local Consul client gets restarted

## Redeploy web
If you `curl` service web from our debug pod, there's no difference

```shell script
kubectl exec debug -- curl -sS web
{
  "name": "Service",
  "uri": "/",
  "type": "HTTP",
  "ip_addresses": [
    "10.1.0.144"
  ],
  "start_time": "2020-05-04T05:25:47.634657",
  "end_time": "2020-05-04T05:25:47.636650",
  "duration": "1.993107ms",
  "body": "Hello World",
  "upstream_calls": [
    {
      "name": "Service",
      "uri": "http://db",
      "type": "HTTP",
      "ip_addresses": [
        "10.1.0.145"
      ],
      "start_time": "2020-05-04T05:25:47.636198",
      "end_time": "2020-05-04T05:25:47.636312",
      "duration": "113.602µs",
      "body": "db",
      "code": 200
    }
  ],
  "code": 200
}
```

But isn't the point that Connect secures our pod?

**📝 Exercise: How is the `web` service
still calling the `db` service?**

<details>
<summary>Hint</summary>

`db` is still listening on all interfaces so `web` can still reach it via
its Kube DNS entry that resolves to its port `9090`. We need to have it
only listen on `localhost` to ensure all traffic must come through the
envoy proxy.

---

</details>

👇

👇

👇

👇

👇

👇

👇

👇

## Secure DB
Luckily Nic's fake-service makes it easy to change the listen address via the `LISTEN_ADDR` env var.

Let's add this to our `db-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
  labels:
    app: db
spec:
  replicas: 2
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
      annotations:
        "consul.hashicorp.com/connect-inject": "true"
    spec:
      containers:
        - name: my-first-container-name
          image: nicholasjackson/fake-service:v0.7.8
          ports:
            - containerPort: 9090
          env:
          - name: MESSAGE
            value: "db"
          - name: LISTEN_ADDR       # <<<<<< ADD THESE
            value: "127.0.0.1:9090" # <<<<<< ADD THESE
```

And redeploy:

```shell script
kubectl apply -f db-deployment.yaml
```

Once the pod comes up, we should be getting an error:

```shell script
kubectl exec debug -- curl -sS web
{
  "name": "Service",
  "uri": "/",
  "type": "HTTP",
  "ip_addresses": [
    "10.1.0.144"
  ],
  "start_time": "2020-05-04T05:30:32.685188",
  "end_time": "2020-05-04T05:30:32.686515",
  "duration": "1.327167ms",
  "upstream_calls": [
    {
      "uri": "http://db",
      "code": -1,
      "error": "Error communicating with upstream service: Get http://db/: dial tcp 10.109.174.100:80: connect: connection refused"
    }
  ],
  "code": 500
}
```

## Get web talking through its envoy proxy

Now we must get `web` talking through it's envoy proxy. We need to
1. Configure it for connect injection
1. Set an upstream
1. Configure it to use that upstream url

**📝 Exercise: How do we do that?**

<details>
<summary>Hint</summary>

Add the annotations:

```yaml
"consul.hashicorp.com/connect-inject": "true"
"consul.hashicorp.com/connect-service-upstreams": "db:8080"
```

And update the `UPSTREAM_URIS` env var to `http://localhost:8080`.

---

</details>

👇

👇

👇

👇

👇

👇

👇

👇

Edit `web.yaml` and add the annotations:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
      annotations:
        "consul.hashicorp.com/connect-inject": "true"
        "consul.hashicorp.com/connect-service-upstreams": "db:8080"
    spec:
      containers:
        - name: my-first-container-name
          image: nicholasjackson/fake-service:v0.7.8
          ports:
            - containerPort: 9090
          env:
          - name: UPSTREAM_URIS
            value: "http://localhost:8080"
```

Redeploy web:

```shell script
kubectl apply  -f web.yaml
deployment.apps/web configured
service/web unchanged
```

Now let's try the `curl`:

```shell script
kubectl exec debug -- curl -sS web
HANGS
```

**📝 Exercise: Uh oh, it's just hanging. What's going wrong? (this is a hard one)**

<details>
<summary>Hint</summary>

If you look at the UI, you'd see that the service names are all weird.
This is because the name we use for our service is the name of the first
container. We need to update our services with an annotation to name them correctly.

---

</details>

👇

👇

👇

👇

👇

👇

👇

👇

## Last time

Update `db-deployment.yaml` and set the `connect-service` annotation:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
  labels:
    app: db
spec:
  replicas: 2
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
      annotations:
        "consul.hashicorp.com/connect-inject": "true"
        "consul.hashicorp.com/connect-service": "db" # <<<< ADD THIS
    spec:
      containers:
        - name: my-first-container-name
          image: nicholasjackson/fake-service:v0.7.8
          ports:
            - containerPort: 9090
          env:
          - name: MESSAGE
            value: "db"
          - name: LISTEN_ADDR
            value: "127.0.0.1:9090"
```

Update `web.yaml` and set the `connect-service` annotation:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
      annotations:
        "consul.hashicorp.com/connect-inject": "true"
        "consul.hashicorp.com/connect-service-upstreams": "db:8080"
        "consul.hashicorp.com/connect-service": "web" # <<<< ADD THIS
    spec:
      containers:
        - name: my-first-container-name
          image: nicholasjackson/fake-service:v0.7.8
          ports:
            - containerPort: 9090
          env:
          - name: UPSTREAM_URIS
            value: "http://localhost:8080"
```

Redeploy the apps:

```shell script
kubectl apply -f db-deployment.yaml -f web.yaml
deployment.apps/db configured
deployment.apps/web configured
service/web unchanged
```

Use the `rollout` command to wait until the deploy is complete:

```shell script
kubectl rollout status deploy/db --watch
deployment "db" successfully rolled out
kubectl rollout status deploy/web --watch
deployment "web" successfully rolled out
```

Now run the curl:

```shell script
kubectl exec debug -- curl -sS web
{
  "name": "Service",
  "uri": "/",
  "type": "HTTP",
  "ip_addresses": [
    "10.1.0.154"
  ],
  "start_time": "2020-05-04T05:45:54.036647",
  "end_time": "2020-05-04T05:45:54.049150",
  "duration": "12.503352ms",
  "body": "Hello World",
  "upstream_calls": [
    {
      "name": "Service",
      "uri": "http://localhost:8080",
      "type": "HTTP",
      "ip_addresses": [
        "10.1.0.152"
      ],
      "start_time": "2020-05-04T05:45:54.048107",
      "end_time": "2020-05-04T05:45:54.048442",
      "duration": "334.549µs",
      "body": "db",
      "code": 200
    }
  ],
  "code": 200
}
```

It should work!

**📝 Exercise:... wait I thought connect was supposed to be secure, how is this allowed?**

<details>
<summary>Hint</summary>

ACLs are turned off so the default policy is allow. If we create a `* => * deny`
intention then the request will fail.

---

</details>

## Next Steps
Nice work! You're ready to move on to the final step: [Canary Deployment](canary-deploy.md)

## Deploy a test application
Most users will already have workloads running in Kubernetes before they look
to adopt a service mesh. We're going to mimic that by installing two applications: web and db.

Create a `db-deployment.yaml` file:

```shell
cat <<EOF > db-deployment.yaml
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
    spec:
      containers:
        - name: my-first-container-name
          image: nicholasjackson/fake-service:v0.7.8
          ports:
            - containerPort: 9090
          env:
          - name: MESSAGE
            value: "db"
EOF
```

This file describes an application called `db`. This is a fake http service that will
respond with the string `"db"` when it gets called at `/`.

Deploy the app to Kubernetes:
```yaml
kubectl apply -f db-deployment.yaml
deployment.apps/db created
```

Check that it created pods:

```shell
kubectl get pod
NAME                                                              READY   STATUS    RESTARTS   AGE
db-7d8985b694-46prc                                               1/1     Running   0          18s
db-7d8985b694-dfjvr                                               1/1     Running   0          75s
```

Before we deploy the `web` service, let's deploy a debug pod that lets us run
commands inside the cluster:

```sh
cat <<EOF > debug.yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug
spec:
  containers:
    - name: debug
      image: anubhavmishra/tiny-tools:latest
      command: ["sleep", "infinity"]
EOF
```

Deploy the debug pod:
```sh
kubectl apply -f debug.yaml
pod/debug created
```

We should be able to `kubectl exec` into the `debug` pod and `curl` the `db` service:
```sh
kubectl exec -it debug sh
curl http://db
curl: (6) Could not resolve host: db
```

Ahh, in Kubernetes, we need to create a `Service` resource that gives our `db` service
a DNS entry. `Ctrl-D` out of the `kubectl exec` or keep it running in another terminal.

Let's create the Service:

```sh
cat <<EOF > db-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  selector:
    app: db
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9090
EOF
```

```sh
kubectl apply -f db-service.yaml
service/db created
```

We can tell it worked by running the `curl` again:

```sh
kubectl exec debug -- curl -s http://db
{
  "name": "Service",
  "uri": "/",
  "type": "HTTP",
  "ip_addresses": [
    "10.1.0.119"
  ],
  "start_time": "2020-05-01T23:14:25.609897",
  "end_time": "2020-05-01T23:14:25.610008",
  "duration": "110.6µs",
  "body": "db",
  "code": 200
}
```

That's better! You can see in `body` that the response is `"db"`.

Now we're ready to deploy our web service:

```shell
cat <<EOF > web.yaml
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
    spec:
      containers:
        - name: my-first-container-name
          image: nicholasjackson/fake-service:v0.7.8
          ports:
            - containerPort: 9090
          env:
          - name: UPSTREAM_URIS
            value: "http://db"
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9090
EOF
```

Note the environment variable `UPSTREAM_URIS`. This configures the `web` service
to use the url for the DB service `http://db` as its dependency. Whenever we make
a call to `web` on `/` it will make a subsequent call to `db`.

Create the deployment and service:

```bash
kubectl apply -f web.yaml 
deployment.apps/web created
service/web created
```

Let's test it again via the `debug` pod:

```shell
kubectl exec debug -- curl -s http://db
{
  "name": "Service",
  "uri": "/",
  "type": "HTTP",
  "ip_addresses": [
    "10.1.0.123"
  ],
  "start_time": "2020-05-01T23:18:30.911220",
  "end_time": "2020-05-01T23:18:30.914587",
  "duration": "3.3657ms",
  "body": "Hello World",
  "upstream_calls": [
    {
      "name": "Service",
      "uri": "http://db",
      "type": "HTTP",
      "ip_addresses": [
        "10.1.0.120"
      ],
      "start_time": "2020-05-01T23:18:30.913281",
      "end_time": "2020-05-01T23:18:30.913677",
      "duration": "396.3µs",
      "body": "db",
      "code": 200
    }
  ],
  "code": 200
}
```

You can see in the `upstream_calls` section of the response that the `web` service
made a call to `http://db`. Our app is working as expected!

## Next Steps
Now you're ready to move onto the next step: [Install Consul](install-consul.md).

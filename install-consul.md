## Installing Consul
We don't like how any service can just call our DB service (remember how we
could call the `db` service from both the `web` and `debug` services?).

We're also migrating to a new version of the db service and we heard that service
meshes are good for that.

To solve these problems, we're going to try out Consul!

## Set up Helm repository

The first step is to add HashiCorp's Helm repository:

```shell script
helm repo add hashicorp https://helm.releases.hashicorp.com/
```

Let's see if we've got the Consul chart available to us now:

```shell script
helm search repo consul
NAME                             	CHART VERSION	APP VERSION	DESCRIPTION
hashicorp/consul                 	0.20.1       	1.7.2      	Official HashiCorp Consul Chart
stable/consul                    	3.9.5        	1.5.3      	Highly available and distributed service discov...
stable/prometheus-consul-exporter	0.1.4        	0.4.0      	A Helm chart for the Prometheus Consul Exporter
```

Hmmm, should we use `hashicorp/consul`? But what's `stable/consul`? Ahh, `hashicorp/consul` is "Official", let's go with that one.

## Create our config file

In order to install a Helm chart, we need to configure it first. We know we
want to enable Consul Connect.

**📝 Exercise: Figure out what your Helm config file should look like (hint: look at our docs).**

<details>
<summary>Hint</summary>

```yaml
# config.yaml
connectInject:
  enabled: true
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

## Install Consul
Do you have your config file ready? **Before** you install, check that it
matches what it should look like:


We also need to add in some config so we only deploy one Consul server since
these Docker environments only have one node and by default each server needs to be on its own node:

```yaml
# config.yaml
server:
  replicas: 1
  bootstrapExpect: 1
```

To install, run:

```shell script
helm install consul hashicorp/consul -f config.yaml
```

## Is it working?

Check out the pods installed by Helm by running

```shell script
kubectl get pod -l app=consul
NAME                                                              READY   STATUS    RESTARTS   AGE
consul-consul-connect-injector-webhook-deployment-7f5db5ffz77tv   1/1     Running   0          104s
consul-consul-mg8xg                                               1/1     Running   0          104s
consul-consul-server-0                                            1/1     Running   0          103s
```

## To the UI!
Let's access the UI to see if everything's working. We can forward a local port
to a running Kubernetes pod:

```shell script
kubectl port-forward consul-consul-server-0 8500:8500
```

If you go to http://localhost:8500 you should see the UI!

## Next Steps
Now you're ready to move onto the next step: [Configure Connect For Our Test App](configure-connect.md).

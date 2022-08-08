# Vault Agent Sidecar Injector Demo

In this demo we are going to walk through a new Vault + kubernetes integration that allows application with no Vault logic built in to source secrets from Vault. This is made possible through a new tool called vault-k8s which leverages the kubernetes mutation admission webhook to intercept and argument (or change) specifically annotated pod definitions for secrets injections.

Note: These are just examples, so **USE AT YOUR OWN RISK**! Please only use with a demo kubernetes cluster when learning. I am assuming you have helm installed and you know how to use Kubernetes already.

Reference Links:

* [Injecting Vault Secrets Into Kubernetes Pods via a Sidecar (blog)](https://www.hashicorp.com/blog/injecting-vault-secrets-into-kubernetes-pods-via-a-sidecar/)
* [Agent Sidecar Injector (docs)](https://www.vaultproject.io/docs/platform/k8s/injector/)
* [Injecting Secrets into Kubernetes Pods via Vault Helm Sidecar (hands on lab)](https://learn.hashicorp.com/vault/getting-started-k8s/sidecar)
* [githup.com/hashicorp/vault-helm](https://github.com/hashicorp/vault-helm)
* [githup.com/hashicorp/vault-k8s](https://github.com/hashicorp/vault-k8s)
* [Video: Vault + Kubernetes Sidecar Injection](https://www.youtube.com/watch?v=xUuJhgDbUJQ)
* [Video: Dynamic Database Credentials with Vault and Kubernetes](https://www.youtube.com/watch?v=KIAXQr17-WQ)

## Clone this demo repo

```shell
git clone https://github.com/jweissig/vault-k8s-sidecar-demo
cd vault-k8s-sidecar-demo
```

## Get the vault-helm git submodule

```shell
git submodule update --init
```

## Configure a GCP k8s cluster

I am going to assume you have a GCP cluster called `vault-cluster` up and running already. Use this command to connect to it.

```shell
gcloud container clusters get-credentials vault-cluster --zone us-central1-a --project sidecar-injection
```

Check to make sure you have nodes ready with `kubectl get nodes`. Also, in a second term window configure a `kubectl get pods` loop like so.

```shell
watch -t -n1 kubectl get pods
```

## Install Vault

Next, lets install Vault onto Kubernetes (running in dev mode).

```shell
kubectl create namespace demo
kubectl config set-context --current --namespace=demo

# helm v3
helm install vault \
       --set='server.dev.enabled=true' \
       ./vault-helm

# or helm v2
helm install --name=vault \
       --set='server.dev.enabled=true' \
       ./vault-helm

```

## Configure Vault

Next, lets configure Vault.

```shell
kubectl exec -ti vault-0 /bin/sh
```

Next, lets configure a policy that we can then attach a role to (used for accessing secrets from a Kubernetes service account).

```shell
cat <<EOF > /home/vault/app-policy.hcl
path "secret*" {
  capabilities = ["read"]
}
EOF
```

Then, lets apply the policy.

```shell
vault policy write app /home/vault/app-policy.hcl
```

Next, enable the kubernetes auth method.

```shell
vault auth enable kubernetes
```

Next, lets configure the kubernetes auth method, so it can communicate with Kubernetes.

```shell
vault write auth/kubernetes/config \
   token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
   kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
   kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

Next, lets connect the policy we created earlier to a myapp role. This will be used in a minute when we create our Kubernetes service account for our demo apps so they are allowed to pull down secrets from Vault.

```shell
vault write auth/kubernetes/role/myapp \
   bound_service_account_names=app \
   bound_service_account_namespaces=demo \
   policies=app \
   ttl=1h
```

Next, lets create some secrets.

```shell
vault kv put secret/helloworld username=foobaruser password=foobarbazpass
vault kv put secret/payment-api APIKEY=497CA26DA27E
vault kv put secret/sendmail-api APIKEY=F5863ABDB85A
```

You can list the secrets by running `vault kv list secret`.

```shell
Keys
----
helloworld
payment-api
sendmail-api
```

Finally, we are done. Lets exit out.

```shell
exit
```

## Running through the demo

Here's a breakdown of what we are going to test:

```shell
app.yaml                          # demo app that knows nothing about vault
patch-basic-annotations.yaml      # sidecar proof of concept
patch-template-annotations.yaml   # now, lets use templates
patch-template-realworld.yaml     # now, lets use multiple secrets
job-pre-populate-only.yaml        # show how jobs work
```

Lets launch our demo app that knows nothing about Vault (plus create the service account).

```shell
vi app.yaml
kubectl create -f app.yaml
kubectl exec -ti app-<POD_NAME> -c app -- ls -l /vault/secrets
```

Next, lets view the definition to check for annotations (there should be none).

```shell
kubectl describe pod app-<POD_NAME> | less
```

Next, lets inject a secret via a sidecar and prove it works.

```shell
vi patch-basic-annotations.yaml
kubectl patch deployment app --patch "$(cat patch-basic-annotations.yaml)"
kubectl exec -ti app-<POD_NAME> -c app -- ls -l /vault/secrets
kubectl exec -ti app-<POD_NAME> -c app -- cat /vault/secrets/helloworld
```

Lets check our annotations again (we should see some).

```shell
kubectl describe pod app-<POD_NAME> | less
```

Next, lets use a template to format the data into something useful.

```shell
vi patch-template-annotations.yaml
kubectl patch deployment app --patch "$(cat patch-template-annotations.yaml)"
kubectl exec -ti app-<POD_NAME> -c app -- ls -l /vault/secrets
kubectl exec -ti app-<POD_NAME> -c app -- cat /vault/secrets/helloworld
```

Next, lets look at a real world example with multiple secrets.

```shell
vi patch-template-realworld.yaml
kubectl patch deployment app --patch "$(cat patch-template-realworld.yaml)"
kubectl exec -ti app-<POD_NAME> -c app /bin/sh
```

Next, lets check out how you might configure a background job.

```shell
vi job-pre-populate-only.yaml
kubectl create -f job-pre-populate-only.yaml
kubectl logs pgdump-
```

## Clean up

Lets delete our demo app.

```shell
kubectl delete deployment app
```

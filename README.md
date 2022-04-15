# jx3-k3s

Jenkins X 3.x GitOps repository using k3s to create a kubernetes cluster, github for the git and container registry and external vault

## Prerequisites

### K3s

Make sure you have created a cluster using k3s.

If you dont have an existing k3s cluster, you can install one by running:

```bash
curl -sfL https://get.k3s.io | sh -
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/k3s-config
```

### Vault

Make sure you have vault running in a docker container with kubernetes auth enabled.

```bash
docker run --cap-add=IPC_LOCK -p 8200:8200 -e 'VAULT_DEV_ROOT_TOKEN_ID=myroot' -e 'VAULT_DEV_LISTEN_ADDRESS=0.0.0.0:8200' --net host vault:latest
```

In another terminal run:

```
export VAULT_ADDR='http://0.0.0.0:8200'
vault auth enable kubernetes
```

## Installation

- Generate a cluster git repository from this template, by clicking [here](https://github.com/ankitm123/jx3-k3s-vault/generate)
- Edit the value of the vault url in the `jx-requirements.yaml` file.
  Replace with `"http://<replace with k3s node name>:8200"`
- Commit and push your changes:

```bash
git add .
git commit -m "fix: set vault url"
git push origin main
```

- Set the GIT_USERNAME and GIT_TOKEN env variable and run:

```bash
jx admin operator --username $GIT_USERNAME --token $GIT_TOKEN --url <url of the cluster git repo> --set "jxBootJobEnvVarSecrets.EXTERNAL_VAULT=\"true\"" --set "jxBootJobEnvVarSecrets.VAULT_ADDR=http://<replace with k3s node name>:8200"
```

> Note:
> The first job will fail as it cannot authenticate against vault. Once the secret-infra namespace has been created, we can configure the kubernetes backend

## Vault configuration

Remember to run the following commands in a terminal where you have set the value of `VAULT_ADDR`

- Create a vault config

```bash
VAULT_HELM_SECRET_NAME=$(kubectl -n secret-infra get secrets --output=json | jq -r '.items[].metadata | select(.name|startswith("kubernetes-external-secrets-token-")).name')
TOKEN_REVIEW_JWT=$(kubectl -n secret-infra get secret $VAULT_HELM_SECRET_NAME --output='go-template={{ .data.token }}' | base64 --decode)
KUBE_CA_CERT=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode)
KUBE_HOST=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.server}')
vault write auth/kubernetes/config \
        token_reviewer_jwt="$TOKEN_REVIEW_JWT" \
        kubernetes_host="$KUBE_HOST" \
        kubernetes_ca_cert="$KUBE_CA_CERT" \
        disable_iss_validation=true
```

- Create a vault role:
```bash
vault write /auth/kubernetes/role/jx-vault bound_service_account_names='*' bound_service_account_namespaces=secret-infra token_policies=jx-policy token_no_default_policy=true disable_iss_validation=true
```

- Create a policy attached to vault role:
```bash
vault policy write jx-policy - <<EOF
path "secret/*" {
  capabilities = ["sudo", "create", "read", "update", "delete", "list"]
}
EOF
```

## Set up ingress and webhook
* To set up webhook, you need to set up ngrok.
# Enable a GitHub Actions workflow to access an OKE cluster using OIDC Authentication

In this example, we authorize a GitHub Actions workflow to run a command or deploy an app in an OKE cluster.

## Step 1 - GitHub repository
Create a new repository in your GitHub account. Write down your account name or organization name, and the name of your repository.

## Step 2 - OKE cluster
Create a Kubernetes cluster with OKE (ref. documentation) or select an existing cluster to update. Write down the cluster OCID.

Then, create a JSON file to update the cluster with the following information:
* isOpenIdConnectAuthEnabled set to True.
* issuerUrl set to "https://token.actions.githubusercontent.com".
* clientId set to the value of your choice. In this example, we use "oke-kubernetes-cluster". This value has to match the audience in your GitHub Actions workflow.
* requiredClaim set to '["repository=GH_ACCOUNT/REPO", "workflow=NAME", "ref=refs/heads/main"]'.
* usernameClaim set to "sub".
* usernamePrefix set to "actions-oidc:".

Replace the GH_ACCOUNT, and REPO with yours. The JSON file is similar to:
``` json
{
    "openIdConnectTokenAuthenticationConfig": {
      "isOpenIdConnectAuthEnabled": true,
      "clientId": "oke-kubernetes-cluster",
      "issuerUrl": "https://token.actions.githubusercontent.com",
      "usernameClaim": "sub",
      "usernamePrefix": "actions-oidc:",
      "requiredClaim": [
        "repository=gregvers/testoidc",
        "workflow=oke-oidc",
        "ref=refs/heads/main"
      ],
      "caCertificate": null,
      "signingAlgorithms": [
        "RS256"
      ]
    }
}
```

Execute the OCI CLI command to update the cluster with your JSON file. Replace the CLUSTER_OCID with yours.
```
oci ce cluster update --cluster-id CLUSTER_OCID --from-json file://./update.json
```

## Step 3 - Egress Network traffic
Allow the cluster API server to egress traffic to GitHub by adding an security rule to your subnet security list of your security rules assigned the cluster Kubernetes API endpoint.

## Step 4 - Kubernetes RBAC
Configure Role Based Access Control (RBAC) in your Kubernetes cluster to authorize your GitHub Actions workflow to do certain operations (get/watch/list pods and get/watch/list/create/update/delete deployments) using the following YAML file. Update the line "name: actions-oidc:repo:GH-ACCOUNT/REPO:ref:refs/heads/main" with your GitHub Account name and Repository name.

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: actions-oidc-role
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "watch", "list", "create", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: actions-oidc-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: actions-oidc-role
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: actions-oidc:repo:GH-ACCOUNT/REPO:ref:refs/heads/main
```

## Step 5 - GitHub Actions
Create an Actions workflow in your GitHub repository: 
* click on the Actions tab,
* click on "Set up a workflow yourself"
* paste the code below in the editor, update server="https://X.X.X.X:6443" with the public IP of your cluster.

```
name: OKE-OIDC

on:
  # Triggers the workflow on push or pull request events but only for the "main" branch
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

permissions:
  id-token: write # Required to receive OIDC tokens

# This workflow generates a GitHub Actions OIDC token and runs kubectl command in an OKE cluster
jobs:
  oke-oidc:
    runs-on: ubuntu-latest
    steps:
      - name: Create OIDC Token
        id: create-oidc-token
        run: |
          AUDIENCE="oke-kubernetes-cluster"
          OIDC_URL_WITH_AUDIENCE="$ACTIONS_ID_TOKEN_REQUEST_URL&audience=$AUDIENCE"
          IDTOKEN=$(curl \
            -H "Authorization: Bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
            -H "Accept: application/json; api-version=2.0" \
            "$OIDC_URL_WITH_AUDIENCE" | jq -r .value)
          echo "::add-mask::${IDTOKEN}"
          echo "idToken=${IDTOKEN}" >> $GITHUB_OUTPUT

      - name: Check Permissions in Kubernetes
        run: |
          kubectl \
          --token=${{ steps.create-oidc-token.outputs.IDTOKEN }} \
          --server="https://X.X.X.X:6443" \
          --insecure-skip-tls-verify \
          auth can-i --list
```

The Github Actions workflow will execute automatically. The kubectl command will return the operations you authorized with the RBAC configuration (get/watch/list pods and get/watch/list/create/update/delete deployments).

## Conclusion
By using GitHub’s native OIDC authentication, you avoid managing long-lived credentials and streamline secure, automated access to your OKE clusters, enabling more efficient CI/CD workflows.


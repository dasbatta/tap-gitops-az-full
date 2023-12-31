# tap-install-azure


- with kube context set to a k8s cluster intended for tap build --> `kapp deploy -a tap-manager -f ./build-app -n tap-install`
- with kube context set to a k8s cluster intended for tap view --> `kapp deploy -a tap-manager -f ./view-app -n tap-install`
- with kube context set to a k8s cluster intended for tap run --> `kapp deploy -a tap-manager -f ./run-app -n tap-install`
- with kube context set to a k8s cluster intended for tap run prod --> `kapp deploy -a tap-manager -f ./run-prod-app -n tap-install`
- with kube context set to a k8s cluster intended for tap full --> `kapp deploy -a tap-manager -f ./full-app -n tap-install`

- kapp deploy -a tap-manager -f ./build-app -n tap-install 

## install eso
helm repo add external-secrets https://charts.external-secrets.io

helm install external-secrets \
   external-secrets/external-secrets \
    -n external-secrets \
    --create-namespace \
    --set installCRDs=true
    
In order to begin using ExternalSecrets, you will need to set up a SecretStore
or ClusterSecretStore resource (for example, by creating a 'vault' SecretStore).

More information on the different types of SecretStores and how to configure them
can be found in our Github: https://github.com/external-secrets/external-secrets

## enable azure workload identity
https://external-secrets.io/v0.8.1/provider/azure-key-vault/#workload-identity

Quickstart on Azure Workload Identity:
https://azure.github.io/azure-workload-identity/docs/quick-start.html

1. Complete the installation guide
Installation guide. At this point, you should have already:

installed the mutating admission webhook
obtained your cluster’s OIDC issuer URL
[optional] installed the Azure AD Workload Identity CLI

```
   export AZURE_TENANT_ID="$(az account show -s <AzureSubscriptionID> --query tenantId -otsv)"
   helm repo add azure-workload-identity https://azure.github.io/azure-workload-identity/charts
   helm repo update
   helm install workload-identity-webhook azure-workload-identity/workload-identity-webhook \
      --namespace azure-workload-identity-system \
      --create-namespace \
      --set azureTenantID="${AZURE_TENANT_ID}"
   curl -sL https://github.com/Azure/azure-workload-identity/releases/download/v1.1.0/azure-wi-webhook.yaml | envsubst | kubectl apply -f -
```


2. Export environment variables
```
   TENANT_ID=$(az account show --query tenantId | tr -d \")
   RESOURCE_GROUP="cssa-resource-group"
   LOCATION="westus"
   VAULT_NAME="tap-eso-vault"
```

> environment variables for the Azure Key Vault resource
```
   export KEYVAULT_NAME="tap-eso-vault"
   export KEYVAULT_SECRET_NAME="example-externalsecret-key"
   export KEYVAULT_SECRET_VAlUE="This is our secret now"
   export RESOURCE_GROUP="cssa-resource-group"
   export LOCATION="westus"
```

> environment variables for the AAD application
```
   export APPLICATION_NAME="eso-app"
```

> environment variables for the Kubernetes service account & federated identity credential
```
   export SERVICE_ACCOUNT_NAMESPACE="tap-install"
   export SERVICE_ACCOUNT_NAME="eso"
   export SERVICE_ACCOUNT_ISSUER="$(az aks show --resource-group <resource_group> --name <cluster_name> --query "oidcIssuerProfile.issuerUrl" -otsv)"
```
3. Create an Azure Key Vault and secret (for initial validation)
```
   az keyvault create --resource-group "${RESOURCE_GROUP}" \
      --location "${LOCATION}" \
      --name "${KEYVAULT_NAME}"

   az keyvault secret set --vault-name "${KEYVAULT_NAME}" \
      --name "${KEYVAULT_SECRET_NAME}" \
      --value ${KEYVAULT_SECRET_VAlUE}"
```

4. Create an AAD application or user-assigned managed identity and grant permissions to access the secret
```
   azwi serviceaccount create phase app --aad-application-name "${APPLICATION_NAME}"
   
   export APPLICATION_CLIENT_ID="$(az ad sp list --display-name "${APPLICATION_NAME}" --query '[0].appId' -otsv)"
   az keyvault set-policy --name "${KEYVAULT_NAME}" \
   --secret-permissions get \
   --spn "${APPLICATION_CLIENT_ID}"
```


5. Create a Kubernetes service account
```
   azwi serviceaccount create phase sa \
   --aad-application-name "${APPLICATION_NAME}" \
   --service-account-namespace "${SERVICE_ACCOUNT_NAMESPACE}" \
   --service-account-name "${SERVICE_ACCOUNT_NAME}"
```

6. Establish federated identity credential between the identity and the service account issuer & subject
```
   azwi serviceaccount create phase federated-identity \
   --aad-application-name "${APPLICATION_NAME}" \
   --service-account-namespace "${SERVICE_ACCOUNT_NAMESPACE}" \
   --service-account-name "${SERVICE_ACCOUNT_NAME}" \
   --service-account-issuer-url "${SERVICE_ACCOUNT_ISSUER}"
```

---
Initial set of secrets seeded in Azure Key Vault to bootstrap kapp (gitops) to load instructions from this repo:
```
   az keyvault secret set --vault-name "${KEYVAULT_NAME}" \
      --name "githubsshknownhosts" \
      -f known_hosts

   az keyvault secret set --vault-name "${KEYVAULT_NAME}" \
      --name "githubsshprivatekey" \
      -f identity

   az keyvault secret set --vault-name "${KEYVAULT_NAME}" \
      --name "githubsshpublickey" \
      -f identity.pub
```

Imperatively applied:
- ./secretstore.yaml
- ./tap-install-gitops-ssh-external-secret.yaml

## Adding additional  notes here, Sept 4th 2023

# With ConnectionString

```
az account set --subscription 87654321-3a70-4c5d-8962-8b51e839a13c	
az servicebus namespace create --name fabmedical-aiw --resource-group fabmedical-aiw --sku basic
az servicebus queue create --namespace-name fabmedical-aiw --name orders --resource-group fabmedical-aiw
az servicebus queue authorization-rule create --resource-group fabmedical-aiw --namespace-name fabmedical-aiw --queue-name orders --name order-consumer --rights Listen
az servicebus queue authorization-rule keys list --resource-group fabmedical-aiw --namespace-name fabmedical-aiw --queue-name orders --name order-consumer
```

"Endpoint=sb://fabmedical-aiw.servicebus.windows.net/;SharedAccessKeyName=order-consumer;SharedAccessKey==;EntityPath=orders"

echo -n "Endpoint=sb://fabmedical-aiw.servicebus.windows.net/;SharedAccessKeyName=order-consumer;SharedAccessKey==;EntityPath=orders" | base64

```
az servicebus queue authorization-rule create --resource-group fabmedical-aiw --namespace-name fabmedical-aiw --queue-name orders --name keda-monitor --rights Manage Send Listen
az servicebus queue authorization-rule keys list --resource-group fabmedical-aiw --namespace-name fabmedical-aiw --queue-name orders --name keda-monitor
```

echo -n "Endpoint=sb://fabmedical-aiw.servicebus.windows.net/;SharedAccessKeyName=keda-monitor;SharedAccessKey=mzssnP0GcLM+/I/JAp9uPg3bqURhw=;EntityPath=orders" | base64

```
kubectl apply -f .\deploy/connection-string/deploy-autoscaling.yaml --namespace keda-dotnet-sample
```

```
dotnet run --project .\src\Keda.Samples.Dotnet.OrderGenerator\Keda.Samples.Dotnet.OrderGenerator.csproj
```

# With Workload Identity

```
az ad app create --display-name keda-fabmedical --native-app --required-resource-accesses @deploy/workload-identity/manifest.json

# appId: 9c4f3781-ed9f-41a2-b8f1-c3fb193c5152
# objectId: 8902ab87-a1fe-4d20-9b9e-b00af63e6610
# tenantId: 12345678-bcf8-4916-a677-b5753051f846

az ad sp create --id 9c4f3781-ed9f-41a2-b8f1-c3fb193c5152
#id/objectid: 34c92094-cfe7-4b6a-93a0-5683014b3337

```

```
az feature register --namespace "Microsoft.ContainerService" --name "EnableWorkloadIdentityPreview"
az provider register --namespace Microsoft.ContainerService
az aks update -g fabmedical-aiw -n fabmedical-aiw --enable-oidc-issuer --enable-workload-identity 
oidcIssuerUrl=$(az aks show --name fabmedical-aiw --resource-group fabmedical-aiw --query oidcIssuerProfile.issuerUrl --output tsv)

az rest --method GET --uri "https://graph.microsoft.com/beta/applications/8902ab87-a1fe-4d20-9b9e-b00af63e6610/federatedIdentityCredentials"

az rest --method POST --uri "https://graph.microsoft.com/beta/applications/8902ab87-a1fe-4d20-9b9e-b00af63e6610/federatedIdentityCredentials" --body @deploy/workload-identity/federate-body.json
```

```
kubectl create namespace keda
helm upgrade keda kedacore/keda \
      --set podIdentity.azureWorkload.enabled=true \
      --set podIdentity.azureWorkload.clientId=9c4f3781-ed9f-41a2-b8f1-c3fb193c5152 \
      --set podIdentity.azureWorkload.tenantId=12345678-bcf8-4916-a677-b5753051f846 \
      --namespace keda
```

```
az role assignment create --role 'Azure Service Bus Data Owner' --assignee 9c4f3781-ed9f-41a2-b8f1-c3fb193c5152 --scope /subscriptions/87654321-3a70-4c5d-8962-8b51e839a13c/resourceGroups/fabmedical-aiw/providers/Microsoft.ServiceBus/namespaces/fabmedical-aiw

```

kubectl apply -f .\deploy\workload-identity\deploy-app-autoscaling.yaml --namespace keda-dotnet-sample
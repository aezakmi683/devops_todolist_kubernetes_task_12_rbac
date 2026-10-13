\# INSTRUCTION.md



\## Kubernetes RBAC for ToDo Application



This document describes how to validate the Kubernetes RBAC configuration.



\### Prerequisites



\- Docker

\- kind

\- kubectl

\- Git



\### 1. Create the Kubernetes cluster



Run the following command from the repository root:



```bash

kind create cluster --config cluster.yml

```



If the cluster already exists, this step can be skipped.



Verify the cluster:



```bash

kubectl get nodes

```



\### 2. Apply RBAC resources



Create the ServiceAccount, Role, and RoleBinding:



```bash

kubectl apply -f security/rbac.yml

```



Verify the resources:



```bash

kubectl get serviceaccount,role,rolebinding -n todoapp

```



The following resources should exist:



\- ServiceAccount: `todoapp-sa`

\- Role: `todoapp-secrets-reader`

\- RoleBinding: `todoapp-secrets-reader-binding`



\### 3. Deploy the application



Apply the application Deployment:



```bash

kubectl apply -f .infrastructure/app/deployment.yml

```



Wait for the rollout to complete:



```bash

kubectl rollout status deployment/todoapp -n todoapp --timeout=120s

```



Verify that the Deployment uses the expected ServiceAccount:



```bash

kubectl get deployment todoapp -n todoapp -o jsonpath="{.spec.template.spec.serviceAccountName}"

```



Expected output:



```text

todoapp-sa

```



\### 4. Verify RBAC permissions



Verify that the ServiceAccount can list Secrets in the `todoapp` namespace:



```bash

kubectl auth can-i list secrets --as=system:serviceaccount:todoapp:todoapp-sa -n todoapp

```



Expected output:



```text

yes

```



Verify that the ServiceAccount cannot create Secrets:



```bash

kubectl auth can-i create secrets --as=system:serviceaccount:todoapp:todoapp-sa -n todoapp

```



Expected output:



```text

no

```



\### 5. Test access to the Kubernetes API from the application Pod



Get the application Pod name:



```bash

kubectl get pods -n todoapp

```



Choose a running Pod and execute the following PowerShell command, replacing `POD\_NAME` with the actual Pod name:



```powershell

$pod = "POD\_NAME"

$token = kubectl exec -n todoapp $pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token

kubectl exec -n todoapp $pod -- curl -sS --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt -H "Authorization: Bearer $token" "https://kubernetes.default.svc/api/v1/namespaces/todoapp/secrets" -o /dev/null -w "HTTP status: %{http\_code}`n"

```



Expected output:



```text

HTTP status: 200

```



HTTP status `200` confirms that the Kubernetes API accepted the request and returned the Secret list.



To display only the Secret names without printing Secret data, run:



```powershell

$pod = "POD\_NAME"

$token = kubectl exec -n todoapp $pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token

$response = kubectl exec -n todoapp $pod -- curl -sS --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt -H "Authorization: Bearer $token" "https://kubernetes.default.svc/api/v1/namespaces/todoapp/secrets"

($response | ConvertFrom-Json).items | ForEach-Object { $\_.metadata.name }

```



\### 6. Evidence



Attach a screenshot to the pull request showing:



\- The successful Kubernetes API request from the application Pod.

\- HTTP status `200` or the returned Secret names.

\- The RBAC permission check returning `yes`.



Do not include Secret values, passwords, ServiceAccount tokens, or other sensitive information in screenshots or pull request comments.




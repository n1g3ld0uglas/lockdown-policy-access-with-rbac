# lockdown-policy-access-with-rbac
Using RBAC to enforce Access Controls on Tiered Network Policies

Token authentication is the default authentication option for Calico Enterprise Manager. 
https://docs.tigera.io/getting-started/cnx/authentication-quickstart

# Create a fully-privleged service account - 'Nigel'

When a service account is created, an associated secret is created that contains a signed bearer token for that service account. 
Just copy the token for the service account in to the Manager UI and log in:

```
kubectl create sa nigel -n default
```

Give the service account 'Nigel' in the default namespace full network admin permissions
```
kubectl create clusterrolebinding nigel-access --clusterrole tigera-network-admin --serviceaccount default:nigel
```

Get the token from the service account 'Nigel'
```
kubectl get secret $(kubectl get serviceaccount nigel -o jsonpath='{range .secrets[*]}{.name}{"\n"}{end}' | grep token) -o go-template='{{.data.token | base64decode}}' && echo
```

If you decided to creating a LoadBalancer service, it may take a few minutes for the load balancer to be created. 
Once complete, the load balancer IP address appears as an 'ExternalIP'. Access this via the below command: 

```
kubectl get services -n tigera-manager tigera-manager-external
```

Access the Calico Enterprise Manager UI in your browser at: https://ExternalIP:9443
https://docs.tigera.io/getting-started/cnx/access-the-manager


# Create a lesser-privleged service account - 'Taher'

The roles and bindings in this file provide a minimum starting point for setting up RBAC
```
wget https://docs.tigera.io/getting-started/kubernetes/installation/hosted/cnx/demo-manifests/min-ui-user-rbac.yaml
```

Run this command to replace with the name or email of the user you are permitting. In this case it's 'Taher'
```
sed -i -e 's/<USER>/taher/g' min-ui-user-rbac.yaml
```

Use the following command to install the bindings
```
kubectl apply -f min-ui-user-rbac.yaml
```

Create a 2nd Service account 'Taher' with limit permissions
```
kubectl create serviceaccount taher
```

Create an Admin user with limited UI access to the Calico Enterprise Manager:

```
kubectl create clusterrolebinding taher-access --clusterrole tigera-ui-user --serviceaccount default:taher
```

Get the token from the service account 'Taher'
```
kubectl get secret $(kubectl get serviceaccount taher -o jsonpath='{range .secrets[*]}{.name}{"\n"}{end}' | grep token) -o go-template='{{.data.token | base64decode}}' && echo
```

# Create a service account with read-only access to all tiers - 'Jessie'

```
kubectl create sa jessie
```

We will give Jessie the same 'tigera'ui-user' cluster role that we assigned to Taher
```
kubectl create clusterrolebinding jessie-access --clusterrole tigera-ui-user --serviceaccount default:jessie
```

Create the 'all-tier-reader' custom role
```
kubectl apply -f https://github.com/n1g3ld0uglas/netpolTest/blob/main/tiers/all-tier-reader.yaml
```

Assign the newly-created cluster role to a binding 'jessie-get':
```
kubectl create clusterrolebinding jessie-get --clusterrole product-tier-policy-cruder --serviceaccount default:jessie
```

ClusterRole is used to provide read access to all policy resource types across all tiers
```
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/netpolTest/main/tiers/all-tier-reader.yaml
```

Assign a newly-create cluster role for read-only access across tiers:
```
kubectl create clusterrolebinding jessie-get --clusterrole all-tier-policy-reader --serviceaccount default:jessie
```

Get the token from the service account 'Jessie'
```
kubectl get secret $(kubectl get serviceaccount jessie -o jsonpath='{range .secrets[*]}{.name}{"\n"}{end}' | grep token) -o go-template='{{.data.token | base64decode}}' && echo
```

# RBAC for Compliance Reports


The following sample manifest creates RBAC for three users: Paul, Candice and David.
1) Paul has permissions to create/modify/delete the report schedules and configuration, but does not have permission to export generated reports from the UI.

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: tigera-compliance-manage-report-config
rules:
- apiGroups: ["projectcalico.org"]
  resources: ["globalreports"]
  verbs: ["*"]
- apiGroups: ["projectcalico.org"]
  resources: ["globalreports/status"]
  verbs: ["get", "list", "watch"]
```

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: tigera-compliance-manage-report-config
subjects:
- kind: User
  name: paul
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: tigera-compliance-manage-report-config
  apiGroup: rbac.authorization.k8s.io
```

2) Candice has permissions to list and export generated reports from the UI, but cannot modify the report schedule or configuration.
```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: tigera-compliance-list-download-all-reports
rules:
- apiGroups: ["projectcalico.org"]
  resources: ["globalreports"]
  verbs: ["get", "list"]
- apiGroups: ["projectcalico.org"]
  resources: ["globalreporttypes"]
  verbs: ["get"]
```

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: tigera-compliance-list-download-all-reports
subjects:
- kind: User
  name: candice
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: tigera-compliance-list-download-all-reports
  apiGroup: rbac.authorization.k8s.io
```

3) David has permissions to list and export generated dev-inventory reports from the UI, but cannot list or download other reports, nor modify the report schedule or configuration.
```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: tigera-compliance-list-download-dev-inventory
rules:
- apiGroups: ["projectcalico.org"]
  resources: ["globalreports"]
  verbs: ["list"]
- apiGroups: ["projectcalico.org"]
  resources: ["globalreports"]
  verbs: ["get"]
  resourceNames: ["dev-inventory"]
- apiGroups: ["projectcalico.org"]
  resources: ["globalreporttypes"]
  verbs: ["get"]
  resourceNames: ["dev-inventory"]
```

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: tigera-compliance-list-download-dev-inventory
subjects:
- kind: User
  name: david
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: tigera-compliance-list-download-dev-inventory
  apiGroup: rbac.authorization.k8s.io
```

# RBAC for Logs & Events

Elasticsearch resources are associated with the Kubernetes API group, lma.tigera.io. 
You can grant access to resources per cluster - Default cluster name for CaliEnt is 'cluster'. 

Each Elasticsearch resource is mapped to a specific RBAC resource name within the lma.tigera.io API group. 
In Calico Enterprise Manager, Elasticsearch resources are called, indexes or indices.



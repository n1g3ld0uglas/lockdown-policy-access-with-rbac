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

# Create a service account with full access to 1 tier - 'Jessie'

# AKS security setup

## Configure API and cluster access

Secure access to to the API and cluster resources using the **Microsoft Entra ID authentication with Azure RBAC** authentication option.

Benefits:

1. Access to the KubeAPI requires Entra (Microsoft AAD) authentication.
2. K8S Roles and role binding can then be configured withing the cluster to give Entra users or Groups necessary permissions.
3. Best practice is to confiure AAD groups rather than individual users and set up clusterrole bindings for those groups. That way individuals can be given specific cluster access using IAM.

## Restrict access to Instance Metadata API

https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-security?tabs=azure-cli#restrict-access-to-instance-metadata-api

Add a network policy in all user namespaces to block pod egress to the metadata endpoint.

> [!NOTE]
> To implement Network Policy, include the attribute --network-policy azure when creating the AKS cluster. Use the following command to create the cluster: az aks create -g myResourceGroup -n myManagedCluster --network-plugin azure --network-policy azure --generate-ssh-keys

K8s manifest:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-instance-metadata
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 10.10.0.0/0#example
        except:
        - 169.254.169.254/32
```

## Limit contianer and pod access to resources

LPA practices need to apply at pod and container level too. They shpould restrict access to filesystem paths and resources. Using a zero trust security module such as AppArmour is a good way of enforcing this.


# KubePlatform

Create production-ready Development Platform for Kubernetes. It contains tools for:

- Monitoring, Alerting, Logging
- Ingress based adding of DNS entries and TLS Certificates
- Oauth based authentication
- CI/CD

In detail - installed tools are:

- nginx Ingress Controller
- Prometheus (+ node-exporter), Grafana, Alert Manager, kube-state-metrics, etc.
- EFK Stack
- External DNS
- cert-manager
- oauth2-proxy
- keycloak
- argo Workflow, argo-events

Visit the [KubePlatform component description](https://kube-platform.github.io/docs/components/) for more insight.

## Precondition
- [kustomize](https://github.com/kubernetes-sigs/kustomize/releases) is needed for the installation
- [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html#installing-eksctl) is needed to install the cluster

## Installation

### Create a Cluster

The current documentation can be found [here](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html)

First step is to create the cluster itself:

```
eksctl create cluster \
 --name event \
 --version 1.16 \
 --without-nodegroup
```

It takes about 10 minutes.

This command will add the new cluster as default to your local kubeconfig so that you can manage it immediately.

This cluster does not contain worker nodes yet. To add those nodes use:

```
eksctl create nodegroup \
--cluster event \
--version auto \
--name standard-workers \
--node-type t3.large \
--node-ami auto \
--nodes 4 \
--nodes-min 2 \
--nodes-max 6
```

This also takes about 10 minutes.

The instances defined above are using a dimension that should be enough to run the kubeplatform infrastructure as well as some of your applications. 

Test whether the cluster is setup correctly by:

```
kubectl get nodes
```

This should show 4 nodes like this:

```
NAME                                           STATUS   ROLES    AGE   VERSION
ip-xxx-xxx-xxx-xxx.eu-west-1.compute.internal   Ready    <none>   20h   v1.16.8-eks-e16311
ip-xxx-xxx-xxx-xxx.eu-west-1.compute.internal   Ready    <none>   20h   v1.16.8-eks-e16311
ip-xxx-xxx-xxx-xxx.eu-west-1.compute.internal   Ready    <none>   20h   v1.16.8-eks-e16311
ip-xxx-xxx-xxx-xxx.eu-west-1.compute.internal   Ready    <none>   20h   v1.16.8-eks-e16311
```

Note: it may happen that the new node group is not displayed in the EKS console.

### Create a DNS hosted zone

Kubeplatform needs a DNS Zone where it can create additional subdomains. This zone needs to be setup in Route53:

```
aws route53 create-hosted-zone --name $(DOMAIN) --caller-reference "$(date)"
```

The result will look something like this:

```
{
    "HostedZone": {
        "ResourceRecordSetCount": 2,
        "CallerReference": "Fr 19 Jun 2020 11:34:17 CEST",
        "Config": {
            "PrivateZone": false
        },
        "Id": "/hostedzone/XXXXXXXXXXXXXXXXXXX",
        "Name": "$(DOMAIN)"
    },
    "DelegationSet": {
        "NameServers": [
            "ns-xxx.awsdns-08.org",
            "ns-xxx.awsdns-54.com",
            "ns-xxx.awsdns-10.net",
            "ns-xxx.awsdns-03.co.uk"
        ]
    },
    "Location": "https://route53.amazonaws.com/2013-04-01/hostedzone/XXXXXXXXXXXXXX",
    "ChangeInfo": {
        "Status": "PENDING",
        "SubmittedAt": "2020-06-19T09:34:18.009Z",
        "Id": "/change/XXXXXXXXXXXXXXXXXXX"
    }
}
```

Make a note of the nameservers that were assigned to your new DNS zone.
Enter the new nameservers in your domain configuration of your domain providers DNS.

Enter the domain name in your kustomization.yaml

### Create a policy for DNS access

First create the policy itself:

```
aws iam create-policy --policy-name kubeplatform-allow-dns --policy-document '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}'
```

Then assign this policy to the role that automatically was created for the cluster.

To find the of the role, display the content of the aws-auth configmap

```
kubectl get configmap -n kube-system aws-auth -o yaml
```

and find the value of the rolearn key. The role name is the part of the ARN following the slash (/). Assign the policy created above to this role:

```
aws iam attach-role-policy --role-name <your-role-name> --policy-arn=<your-policy-arn>
```

*Caution* This gives all pods admin access to Route53. Do not use this in production environments and be careful about the pods you deploy to any clusters you set up this way.

TODO Find better solution to handle DNS updates

### Install kubeplatform inside cluster

Create a namespace for kubeplattform. We usually use 'kubeplatform', if you want to chose an other you need to update your kustomization.yaml.

```
kubectl create namespace ${NAMESPACE}
```

Then apply the kustomize overlay to the cluster:

```
kubeclt apply -k <path-to-aws-eks-overlay>
```

If you encounter error messages about 'no matches for kind "ClusterIssuer"' that means the operator for the corresponding CRD has not kicked in yet. Simply wait a few moments and run the command again.

### Finalize

Wait until your Pods are running:

```
kubectl get pods --namespace $(NAMESPACE)
```

Setup a user in Keycloak:

A call to https://keycloak.$(DOMAIN)/auth/admin/ should point you to your Keycloak instance (username is keycloak, for the password refer to your kustomization.yaml)
Add a user of your choice in Manage/Users (must have an email address). Please refer to the respective Keycloak documentation
You should then be able to use this user to go to:

* https://prometheus.$(DOMAIN)
* https://kibana.$(DOMAIN)
* https://grafana.$(DOMAIN)
* https://argo.$(DOMAIN)

If you want to see the internal metrics collected by Prometheus, you can start by importing a Kubernetes dashboard into Grafana, e.g. https://grafana.com/grafana/dashboards/6417

If you want to remove all KubePlatform resources from the cluster, simplly use the following command:

```
kubectl delete -k aws-eks-overlay

kubectl --namespace $NAMESPACE delete secret letsencrypt-staging
kubectl --namespace $NAMESPACE delete secret letsencrypt-prod

kubectl --namespace $NAMESPACE delete secret grafana-tls
kubectl --namespace $NAMESPACE delete secret keycloak-tls
kubectl --namespace $NAMESPACE delete secret kibana-logging-tls
kubectl --namespace $NAMESPACE delete secret prometheus-tls

kubectl --namespace $NAMESPACE delete secret cert-manager-webhook-tls
kubectl --namespace $NAMESPACE delete secret cert-manager-webhook-ca

kubectl --namespace $NAMESPACE delete cm cert-manager-controller
kubectl --namespace $NAMESPACE delete cm ingress-controller-leader-nginx
```
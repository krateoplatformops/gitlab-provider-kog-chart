# GitLab Provider Helm Chart

This is a [Helm Chart](https://helm.sh/docs/topics/charts/) that deploys the Krateo GitLab Provider leveraging the [Krateo OASGen Provider](https://github.com/krateoplatformops/oasgen-provider) and using OpenAPI Specifications (OAS) of the GitLab API.
This provider allows you to manage GitLab resources such as repositories.

## Summary

- [Summary](#summary)
- [Requirements](#requirements)
- [How to install](#how-to-install)
- [Supported resources](#supported-resources)
  - [Resource details](#resource-details)
    - [Repo](#repo)
  - [Resource examples](#resource-examples)
- [Authentication](#authentication)
- [Configuration](#configuration)
  - [Verbose logging](#verbose-logging)
- [Chart structure](#chart-structure)
- [Troubleshooting](#troubleshooting)

## Requirements

[Krateo OASGen Provider](https://github.com/krateoplatformops/oasgen-provider) should be installed in your cluster. Follow the related Helm Chart [README](https://github.com/krateoplatformops/oasgen-provider-chart) for installation instructions.

## How to install

To install the chart, use the following commands:

```sh
helm repo add krateo https://charts.krateo.io
helm repo update krateo
helm install gitlab-provider krateo/gitlab-provider-kog
```

> [!NOTE]
> Due to the nature of the providers leveraging the [Krateo OASGen Provider](https://github.com/krateoplatformops/oasgen-provider), this chart will install a set of RestDefinitions that will in turn trigger the deployment of controllers in the cluster. These controllers need to be up and running before you can create or manage resources using the Custom Resources (CRs) defined by this provider. This may take a few minutes after the chart is installed.

You can check the status of the controllers by running:
```sh
until kubectl get deployment gitlab-provider-<RESOURCE>-controller -n <YOUR_NAMESPACE> &>/dev/null; do
  echo "Waiting for <RESOURCE> controller deployment to be created..."
  sleep 5
done
kubectl wait deployments gitlab-provider-<RESOURCE>-controller --for condition=Available=True --namespace <YOUR_NAMESPACE> --timeout=300s
```

Make sure to replace `<RESOURCE>` to one of the resources supported by the chart, such as `repo`, and `<YOUR_NAMESPACE>` with the namespace where you installed the chart.

For instance, in the case of the `Repo` resource, you would run:
```sh
until kubectl get deployment gitlab-provider-repo-controller -n <YOUR_NAMESPACE> &>/dev/null; do
  echo "Waiting for Repo controller deployment to be created..."
  sleep 5
done
kubectl wait deployments gitlab-provider-repo-controller --for condition=Available=True --namespace <YOUR_NAMESPACE> --timeout=300s
```

## Supported resources

This chart supports the following resources and operations:

| Resource     | Get  | Create | Update | Delete |
|--------------|------|--------|--------|--------|
| Repo         | ✅   | ✅     | ✅     | ✅     |


The resources listed above are Custom Resources (CRs) defined in the `gitlab.kog.krateo.io` API group. They are used to manage GitLab resources in a Kubernetes-native way, allowing you to create, update, and delete GitLab resources using Kubernetes manifests.

### Resource details

#### Repo

The `Repo` resource allows you to create, update, and delete GitLab repositories.

An example of a Repo resource is:
```yaml
apiVersion: gitlab.kog.krateo.io/v1alpha1
kind: Repo
metadata:
  name: test-repo
  namespace: glp
spec:
  authenticationRefs:
    bearerAuthRef: bearer-gitlab
  name: test-repo
```

### Resource examples

You can find example resources for each supported resource type in the `/samples` folder of the chart.
These examples Custom Resources (CRs) shows every possible field that can be set in the resource.

## Authentication

The authentication to the GitLab API is managed using 2 resources (both are required):

- **Kubernetes Secret**: This resource is used to store the GitLab Token that is used to authenticate with the GitLab API. The Token should have the necessary permissions to manage the resources you want to create or update.

Example of a Kubernetes Secret that you can apply to your cluster:
```sh
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-repo-creds 
  namespace: gitlab-system
type: Opaque
stringData:
  token: <TOKEN>
EOF
```

Replace `<TOKEN>` with your actual GitLab Token.

- **BearerAuth**: This resource references the Kubernetes Secret and is used to authenticate with the GitLab API. It is used in the `authenticationRefs` field of the resources defined in this chart.

Example of a BearerAuth resource that references the Kubernetes Secret, to be applied to your cluster:
```sh
kubectl apply -f - <<EOF
apiVersion: gitlab.kog.krateo.io/v1alpha1
kind: BearerAuth
metadata:
  name: bearer-gitlab
  namespace: glp
spec:
  tokenRef:
    key: token
    name: gitlab-repo-creds 
    namespace: gitlab-system 
EOF
```

## Configuration

### Verbose logging

In order to enable verbose logging for the controllers, you can add the `krateo.io/connector-verbose: "true"` annotation to the metadata of the resources you want to manage, as shown in the examples above. 
This will enable verbose logging for those specific resources, which can be useful for debugging and troubleshooting.

## Chart structure

Main components of the chart:

- **RestDefinitions**: These are the core resources needed to manage resources leveraging the Krateo OASGen Provider. In this case, they refers to the OpenAPI Specification to be used for the creation of the Custom Resources (CRs) that represent GitLab resources.
They also define the operations that can be performed on those resources. Once the chart is installed, RestDefinitions will be created and as a result, specific controllers will be deployed in the cluster to manage the resources defined with those RestDefinitions.

- **ConfigMaps**: Refer directly to the OpenAPI Specification content in the `/assets` folder.

- **/assets** folder: Contains the selected OpenAPI Specification files for the GitLab API.

- **/samples** folder: Contains example resources for each supported resource type as seen in this README. These examples demonstrate how to create and manage GitLab resources using the Krateo GitLab Provider.

## Troubleshooting

For troubleshooting, you can refer to the [Troubleshooting guide](./docs/troubleshooting.md) in the `/docs` folder of this chart. It contains common issues and solutions related to this chart.

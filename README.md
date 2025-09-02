# GitLab Provider KOG Helm Chart

***KOG***: (*Krateo Operator Generator*)

This is a [Helm Chart](https://helm.sh/docs/topics/charts/) that deploys the Krateo GitLab Provider leveraging the [Krateo OASGen Provider](https://github.com/krateoplatformops/oasgen-provider) and using OpenAPI Specifications (OAS) of the GitLab API.
This provider allows you to manage GitLab resources such as repositories.

## Summary

- [Requirements](#requirements)
- [How to install](#how-to-install)
- [Supported resources](#supported-resources)
  - [Resource details](#resource-details)
    - [Repo](#repo)
  - [Resource examples](#resource-examples)
- [Authentication](#authentication)
- [Configuration](#configuration)
  - [Configuration resources](#configuration-resources)
  - [Verbose logging](#verbose-logging)
- [Chart structure](#chart-structure)

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
kubectl get restdefinitions.ogen.krateo.io --all-namespaces | awk 'NR==1 || /gitlab/'
```

You should see output similar to this:
```sh
NAMESPACE       NAME                           READY   AGE
krateo-system   gitlab-provider-repo           False   24s
```

You can also wait for a specific RestDefinition (`gitlab-provider-repo` in this case) to be ready with a command like this:
```sh
kubectl wait restdefinitions.ogen.krateo.io gitlab-provider-repo --for condition=Ready=True --namespace krateo-system --timeout=300s
```

Note that the names of the RestDefinitions and the namespace where the RestDefinitions are installed may vary based on your configuration.

## Supported resources

This chart supports the following resources and operations:

| Resource     | Get  | Create | Update | Delete |
|--------------|------|--------|--------|--------|
| Repo         | ✅   | ✅     | ✅     | ✅     |

The resources listed above are Custom Resources (CRs) defined in the `gitlab.ogen.krateo.io` API group. They are used to manage GitLab resources in a Kubernetes-native way, allowing you to create, update, and delete GitLab resources using Kubernetes manifests.

### Resource details

#### Repo

The `Repo` resource allows you to create, update, and delete GitLab repositories.

Note: currently only the creation of repositories in the user's personal namespace (default namespace) is supported.

An example of a Repo resource is:
```yaml
apiVersion: gitlab.ogen.krateo.io/v1alpha1
kind: Repo
metadata:
  name: test-repo
  namespace: default
spec:
  configurationRef:
    name: my-gitlab-repo-config
    namespace: default 
  name: test-repo
  visibility: public
  initialize_with_readme: true
```

The `Repo` resource schema includes the following fields:

| Field | Type | Description | Notes |
| :--- | :--- | :--- | :--- |
| `configurationRef.name` | `string` | Name of the RepoConfiguration resource to use. |
| `configurationRef.namespace` | `string` | Namespace of the RepoConfiguration resource to use. |
| `name` | `string` | Name of the repository. | Required. |
| `initialize_with_readme` | `boolean` | Whether to initialize the repository with a README file. | Optional. Default: `false`. |
| `visibility` | `string` | Visibility level of the repository. | Optional. One of: `private`, `internal`, `public`. Default: `private`. |

### Resource examples

You can find example resources for each supported resource type in the `/samples` folder of the chart.

## Authentication

The authentication to the GitLab API is managed using 2 resources (both are required):

- **Kubernetes Secret**: This resource is used to store the GitLab Token that is used to authenticate with the GitLab API. The Token should have the necessary permissions to manage the resources you want to create or update.

Example of a Kubernetes Secret that you can apply to your cluster:
```sh
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-token
  namespace: default
type: Opaque
stringData:
  token: <TOKEN>
EOF
```

Replace `<TOKEN>` with your actual GitLab Token.

- **\<Resource\>Configuration**: These resource can reference the Kubernetes Secret and are used to authenticate with the GitHub REST API. They must be referenced with the `configurationRef` field of the resources defined in this chart. The configuration resource can be in a different namespace than the resource itself.

Note that the specific configuration resource type depends on the resource you are managing. For instance, in the case of the `Repo` resource, you would need a `RepoConfiguration`.

An example of a `RepoConfiguration` resource that references the Kubernetes Secret, to be applied to your cluster:
```sh
kubectl apply -f - <<EOF
apiVersion: gitlab.ogen.krateo.io/v1alpha1
kind: RepoConfiguration
metadata:
  name: my-gitlab-repo-config
  namespace: default
spec:
  authentication:
    bearer:
      # Reference to a secret containing the bearer token
      tokenRef:
        name: gitlab-token        # Name of the secret
        namespace: default        # Namespace where the secret exists
        key: token                # Key within the secret that contains the token
EOF
```

Then, in the `Repo` resource, you can reference the `RepoConfiguration` resource as follows:
```yaml
apiVersion: gitlab.kog.krateo.io/v1alpha1
kind: Repo
metadata:
  name: test-repo
  namespace: default
spec:
  configurationRef:
    name: my-gitlab-repo-config
    namespace: default
  name: test-repo
  visibility: public
  initialize_with_readme: true
```

More details about the configuration resources in the [Configuration resources](#configuration-resources) section below.

## Configuration

### Configuration resources

Each resource type (e.g., `Repo`) requires a specific configuration resource (e.g., `RepoConfiguration`) to be created in the cluster.
Currently, the supported configuration resources are:
- `RepoConfiguration`

These configuration resources are used to store the authentication information (i.e., reference to the Kubernetes Secret containing the GitLab token) and other configuration options for the resource type.
You can find examples of these configuration resources in the `/samples/configs` folder of the chart.
Note that a single configuration resource can be used by multiple resources of the same type.
For example, you can create a single `RepoConfiguration` resource and reference it in multiple `Repo` resources.

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

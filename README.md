# helm

Copied from : Tamal Saha

Email: [tamal@appscode.com](mailto:tamal@appscode.com), Twitter: @tsaha

## Helm++

Link: [https://appsco.de/helm-wishlist](https://appsco.de/helm-wishlist)

**[Motivation](#_azrc524wfw3s) 2**

**[Ideas](#_rho324ibdug4) 2**

[Use Application CRD](#_oj7fuk8f37u) 2

[Helm Universal Controller for Kubernetes](#_5e2373fotglg) 3

[Use deploy instead of install/upgrade/rollback](#_iy7nn15waih4) 3

[Stop storing release history in secrets](#_2th7cauqvpdh) 3

[Replace Chart.yaml with Chart CRD](#_2g7e9nyantzb) 4

[Use OpenAPI v3 as Chart Values Schema](#_bwrulq7cfbj4) 4

[Using Steps and Waits](#_4326ep6op1q9) 5

[Handling User Defined Resource Types](#_5r7k4p98l57l) 7

[Uninstall vs Purge](#_j8ldy0xwe5p6) 7

[Remove Hooks](#_itx8dbv4ox9l) 7

[Late Binding Values](#_d1znt8gxb5vt) 8

[Exported Values Map](#_ddblm82g56jj) 8

[Starlark Script](#_rjr7jx2gqef3) 9

[Why?](#_gqrdyenqj8bv) 9

[Using Values file from inside a Chart](#_1fzm62qxj239) 9

[Deprecating Chart and Upgrade Recommendation](#_iys8ke6lvmvg) 10

[Chart Doc Generation](#_8t6two6na3cz) 10

[Helm++ CLI](#_tgg3q0jexyzz) 10

###


### Motivation

Helm has won the user mindshare for application packing in Kubernetes. I first started using Helm back in Nov 2016. Since then Helm has come a long way but at its core it has remained mostly the same in terms of core features (with the exception of Tiller removal in Helm 3). From what I can tell this is intentional by design by the core maintainers of Helm.

During this period, there have been various changes in container orchestration. Kubernetes has won the container orchestration war. The other key phenomenon is the rise of CRDs. We are starting to see projects like [Kubeform](https://kubeform.com/), [Google Cloud Config Connector](https://cloud.google.com/config-connector/docs/overview) , [AWS Service Operator](https://github.com/aws/aws-service-operator-k8s) , [Crossplane](https://crossplane.io/) that allows users to manage your Non-Kubernetes (aka cloud) resources through Kubernetes configuration (CRD).

So, I believe there is room to reimagine what Helm could look like if it was designed in 2020 knowing what we know today. So, I am writing down my thoughts in this document and calling it project Helm++.

### Ideas

## Use Application CRD

Kubernetes &quot;is centered on containerized applications. Yet, the Kubernetes metadata, objects, and visualizations (e.g., within Dashboard) are focused on container infrastructure rather than the applications themselves.&quot;
# 1
 . Given Helm has become the standard for installing pre-packages applications in Kubernetes, it makes sense to create an Application CRD object when `helm install` is run. This will enable:

1. Provide a standard API for creating, viewing, and managing applications in Kubernetes.
2. Provide a CLI implementation, via kubectl, that interacts with the Application API.
3. Provide installation status and garbage collection for applications.
4. Provide a standard way for applications to surface a basic health check to the UIs.
5. Provide an explicit mechanism for applications to declare dependencies on another application.
6. Promote interoperability among ecosystem tools and UIs by creating a standard that tools MAY implement.
7. Promote the use of common labels and annotations for Kubernetes Applications.

Now, one of the issues with Kubernetes Application CRD is that it is agnostic to the existence of Helm making it unusable in writing a generic operator. We have added a new field to our version of Application CRD that keeps track of the Chart repository that was used to deploy this &quot;application&quot;. You can see it here:

[https://github.com/kubepack/kubepack/blob/aeca27630e884b120c09dd84844074129eb6d1f5/apis/kubepack/v1alpha1/application\_types.go#L83](https://github.com/kubepack/kubepack/blob/aeca27630e884b120c09dd84844074129eb6d1f5/apis/kubepack/v1alpha1/application_types.go#L83)

## Helm Universal Controller for Kubernetes

There is an ongoing effort in Kubernetes upstream called [kubernetes-sigs/cluster-addons: Addon operators for Kubernetes clusters.](https://github.com/kubernetes-sigs/cluster-addons) This project adds a custom operator for each application that is deployed to Kubernetes. I think this approach is unmanageable at scale and pushing the cost of running an operator per application deployed to end users is not great. This problem can be easily solved by writing a Helm operator (more appropriately an Application CRD controller) which used Helm charts to deploy applications on a cluster. An operator like this can provide installation status and garbage collection for applications and provide a standard way for applications to surface a basic health check to the UIs.

## Use deploy instead of install/upgrade/rollback

Currently Helm differentiates among install / upgrade / rollback verbs which to the Kubernetes api server all of these are &quot;deploy&quot; operations. I think Helm should just use the &quot;deploy&quot; verb instead of 3 separate verbs. This deploy verb should work like &quot;kubectl apply&quot;. This removes a number of complexities around hooks, crds and cross chart resources. This is explained further in the following sections.

Helm++ deploy command will also track the user who performed the deployment using [https://github.com/kubeshield/identity-server](https://github.com/kubeshield/identity-server) (whoami service) and track that in the Application CRD.

&quot;deploy&quot; command should be implemented using [server side apply](https://kubernetes.io/docs/reference/using-api/api-concepts/#server-side-apply) once it is GA. Based on historical progress on this feature and our intention to maintain backward compatibility for 4-6 releases of Kubernetes, this will not be usable generally until 2022 .

## Stop storing release history in secrets

Currently Helm stores release history in secrets. I think Release history should be maintained outside a Kubernetes cluster and outlive the life time of a cluster as it is a source of audit record. Using Git is increasingly becoming a popular choice to store release history. Helm++ project does not mandate any particular storage option as long as it is not stored inside the Kubernetes cluster.

I think the Application CRD should store the Values patch that was used to deploy an application. You can see it here:

[https://github.com/kubepack/kubepack/blob/helm%2B%2B/apis/kubepack/v1alpha1/application\_types.go#L334-L342](https://github.com/kubepack/kubepack/blob/helm%2B%2B/apis/kubepack/v1alpha1/application_types.go#L334-L342)

## Replace Chart.yaml with Chart CRD

Instead of Chart.yml use Chart CRD. If Helm was started today, it would have used CRD instead of a custom YAML format. Here is an example:

[https://github.com/kubepack/kubepack/blob/helm%2B%2B/apis/kubepack/v1alpha1/chart\_types.go#L36](https://github.com/kubepack/kubepack/blob/helm%2B%2B/apis/kubepack/v1alpha1/chart_types.go#L36)

## Use OpenAPI v3 as Chart Values Schema

Helm 3 added support for validating values file using json schema. The issue is that Values files can be complex and long. It is very difficult to write and maintain the json schema by hand. There is a tool that can generate schema from Values file [https://github.com/karuppiah7890/helm-schema-gen](https://github.com/karuppiah7890/helm-schema-gen) . The issue is that this is fundamentally impossible to generate a complete schema for Values file from looking at the default values. This will be akin to generating a definition of a &quot;class&quot; from a single object of that class. This complex becomes particularly obvious when the values file include complex fields like resources, tolerations, etc.

We avoid this problem by reusing existing tools for CRD like [Kubebuilder](https://github.com/kubernetes-sigs/kubebuilder). We can write a CRD whose &quot;spec&quot; defines the Values file of a chart. Then we can use the Kubebuilder tool to generate the CRD YAML which includes the schema in OpenAPI v3 format. We use the &quot;yq&quot; tool to extract the openapi v3 schema for the spec and include that as **values.openapiv3\_schema.yaml** file in a Chart. This approach allows us to generate complete schema for complex fields like tolerations, resources, affinity, securityContext as we are able to use the k8s.io/api objects to define the Values CRD.

Here is an example in our Stash project:

[https://github.com/stashed/installer/blob/cc55e5a52a5ebccc2ef23e7b3eb1237008f0c23f/apis/installer/v1alpha1/stash\_operator\_types.go#L86-L93](https://github.com/stashed/installer/blob/cc55e5a52a5ebccc2ef23e7b3eb1237008f0c23f/apis/installer/v1alpha1/stash_operator_types.go#L86-L93)

[https://github.com/stashed/installer/blob/cc55e5a52a5ebccc2ef23e7b3eb1237008f0c23f/Makefile#L226-L229](https://github.com/stashed/installer/blob/cc55e5a52a5ebccc2ef23e7b3eb1237008f0c23f/Makefile#L226-L229)

[https://github.com/stashed/installer/blob/master/charts/stash/values.openapiv3\_schema.yaml](https://github.com/stashed/installer/blob/master/charts/stash/values.openapiv3_schema.yaml)

One issue is that OpenAPI v3 schema is incompatible with json schema draft 07. But given OpenAPI v3 schema used with CRD is Kubernetes project itself, this approach makes a lot of sense to us.

Also, [OpenAPI v3.1 and JSON Schema 2019-09](https://apisyouwonthate.com/blog/openapi-v31-and-json-schema-2019-09) addresses the compatibility issues. So, we can hope someday Kubernetes will start using OpenAPI v3.1 schema and this issue will be moot.

## Using Steps and Waits

Currently Helm install / upgrade commands render YAMLs from Chart templates, [sorts them](https://github.com/helm/helm/blob/v3.2.1/pkg/releaseutil/kind_sorter.go#L25-L104) and essentially runs a `kubectl apply -f `. This approach does not work well for CRDs because Helm can&#39;t possibly know about CRDs that are available in a Kubernetes cluster. I propose a simple approach, I call &quot;steps&quot;, to address this issue. Chart authors can define a sequence of groups of YAMLs that can be deployed during helm install / upgrade in Chart.yaml file.

Helm install / upgrade / rollback commands has a --wait flag that &quot;waits until all Pods are in a ready state, PVCs are bound, Deployments have minimum (Desired minus maxUnavailable) Pods in ready state and Services have an IP address (and Ingress if a LoadBalancer) before marking the release as successful. It will wait for as long as the --timeout value. If timeout is reached, the release will be marked as FAILED.&quot;
# 2
 This is great but inadequate when CRDs are used as helm is oblivious to CRDs. We think a better approach will be to let Chart authors define wait conditions for each step of the &quot;deploy&quot; process. This will essentially provide the flags used in an equivalent &quot;kubectl wait&quot; command.

Below is an example how this can be defined in Chart.yaml:

| **Chart.yaml**
apiVersion: v2name: chart-stepsdescription: A Helm chart for Kubernetestype: applicationversion: 0.1.0appVersion: 0.1.0steps:- name: install crds **templates:** - step-1/\ ***waitFors:** # kubectl wait --for=condition=NamesAccepted crd -l &#39;app=kubepack&#39; --timeout=5m- resource:group: apiextensions.k8s.ioresource: customresourcedefinitionslabels:matchLabels:app: myapptimeout: 5mfor: &quot;condition=NamesAccepted&quot;- name: deploy operator **templates:** - step-2/\ ***waitFors:** # kubectl wait --for=condition=Ready pods -l app=myapp --timeout=5m- resource:group: appsresource: deploymentslabels:matchLabels:app: myapptimeout: 5mfor: &quot;condition=Ready&quot;
 |
| --- |

| **Chart directory structure**
 ├── Chart.yaml ├── **templates** │ ├── \_helpers.tpl │ ├── NOTES.txt│ ├── **step-1** │ │ ├── kubepack.com\_plans.yaml│ │ └── kubepack.com\_products.yaml│ ├── **step-2** │ │ ├── deployment.yaml│ │ ├── hpa.yaml│ │ ├── ingress.yaml│ │ ├── serviceaccount.yaml│ │ └── service.yaml │ └── tests │ └── test-connection.yaml ├── values.openapiv3\_schema.yaml └── values.yaml
4 directories, 13 files |
| --- |

## Handling User Defined Resource Types

Kubernetes users can define new resource types using CRD or Extender API Server. Chart authors should be able to declare _ **required resource types for a chart** _. This should be checked as a precondition for &quot;deploy&quot; operation. Chart authors should be able to declare resource types introduced by a chart. We say that those _ **types are &quot;owned&quot; by this chart** _.

Helm++ will be able to handle the following scenarios for a CRD

- Registered as part of Helm chart
- Registered by the application that is deployed as Helm chart (side-effect)
- Leave CRD in place when the chart is uninstalled because there may be custom resources related to that CRD.
- Leave the user facing roles for a CRD intact when a chart is installed.

## Uninstall vs Purge

Helm++ will support two modes of &quot;delete&quot; operation: Delete/Uninstall and Uninstall with Purge. The &quot;uninstall&quot; command will work as it works for Helm today. Users will be able to tell Helm++ to [not delete a resource](https://helm.sh/docs/howto/charts_tips_and_tricks/#tell-helm-not-to-uninstall-a-resource) during uninstall command by using the following annotation : **helm.sh/resource-policy&quot;: keep**.

Purge will behave like &quot;uninstall&quot;. In addition, &quot;purge&quot; will also delete:

- Resources that has **&quot;helm.sh/resource-policy&quot;: keep**
- Find and delete resources of types owned by a chart.
- Remove CRDs owned by a chart.

## Remove Hooks

Helm++ will remove the hook feature oh Helm. The &quot;deploy&quot; command, &quot;steps&quot; feature and **helm.sh/resource-policy&quot;: keep** annotation will allow us to handle the use-cases where I find myself using the hook feature today. Hooks is a crude implementation of the hooks feature. _ **If you can think of use-cases that can&#39;t be handled without hooks in Helm++, please let me know in the comments or via email.** _

## Late Binding Values

Currently Helm chart Values are statically defined. This is very limiting. Helm++ will support late binding of values from a cluster. Essentially chart authors will be able to define &quot; **exported values**&quot; for each &quot; **step**&quot; in the Chart.yaml. These exported values will be merged into the existing Values file and will be used to render the resource YAMLs for the next step.

Chart authors will be able to define the exported values in 2 ways:

### Exported Values Map

Define the values exported using a map of key, value pairs for any resource in the cluster. Both key and value paths will be defined using [json pointers](https://tools.ietf.org/html/rfc6901). This can look like:

| apiVersion: v2name: chart-stepsdescription: A Helm chart for Kubernetestype: applicationversion: 0.1.0appVersion: 0.1.0steps:- name: install crdstemplates:- step-1/\*waitFors:# kubectl wait --for=condition=NamesAccepted crd -l &#39;app=kubepack&#39; --timeout=5m- resource:group: apiextensions.k8s.ioresource: customresourcedefinitionslabels:matchLabels:app: myapptimeout: 5mfor: &quot;condition=NamesAccepted&quot; **exportValues:** - apiVersion: apiextensions.k8s.io/v1kind: CustomResourceDefinitionname: mycrd.mygroup.comvalues:- key: .crd.namevalue: .metadata.name |
| --- |

### Starlark Script

In this case, chart authors will be able to define a function that returns a map of keys and values that are output from a [Starlark script](https://github.com/google/starlark-go). I am not sure this level of expressiveness will be really necessary but this is certainly doable. The key to note here is that starlark scripts will be used to enable complex transformations of the input resource yaml only.

### Why?

Late binding of Values will enable a number of use-cases for Helm++:

- This will essentially allow templating in Values file, a requested feature in Helm:
  - [https://github.com/helm/helm/issues/2492](https://github.com/helm/helm/issues/2492)
  - [https://github.com/helm/helm/issues/2133](https://github.com/helm/helm/issues/2133)

- Late binding with exported values map and &quot;deploy&quot; command will give Helm++ the same capabilities of a [Terraform Module](https://www.terraform.io/docs/modules/index.html). This is a requested feature for [Kubeform](https://github.com/kubeform/kubeform) project.

- Currently if you are using admission webhooks in Helm charts, a certificate is dynamically generated every time helm install/upgrade/rollback is done. This is not great from a security perspective because these auto-generated certificates are hard to audit. Late binding will allow chart authors the option to use a project like [Cert Manager](https://github.com/jetstack/cert-manager) to generate certificates and inject those certificates into the Values file for rendering resource YAMLs. It can potentially look like this:
  - step-1: create certificates, wait for secrets, export certs from secrets into Values
  - Step-2: deploy application rendered using updated Values.

## Using Values file from inside a Chart

Helm charts can contain multiple values files. But helm install --values | -f file command can&#39;t values file from inside a chart archive. This file can be loaded from local filesystem or remote urls. I propose that the following syntax to load values file from inside a chart archive.

--values chart:///values-production.yaml

## Deprecating Chart and Upgrade Recommendation

Chart CRD gives us a way to inform users when a new version of a chart is available, since Helm charts follow Semver. This feature is called [channels](https://github.com/kubernetes-sigs/cluster-addons/tree/master/coredns/channels) in addon-operators project.

## Chart Doc Generation

Since charts are used by end users of an application, it is important to have an up-to-date documentation for a chart. [https://github.com/kmodules/chart-doc-gen](https://github.com/kmodules/chart-doc-gen) can be used to generate documentation for a chart. This tool requires some additional information to render the documentation, as seen here:

[https://github.com/kmodules/chart-doc-gen/blob/master/testdata/doc.yaml](https://github.com/kmodules/chart-doc-gen/blob/master/testdata/doc.yaml)

In Helm++, Chart.yaml will include all the data necessary for doc generation.

## Helm++ CLI

We will need a cli for Helm++ project . This will be a kubectl plugin instead of top level cli like Helm. Here are some ideas for command structure:

| $ kubectl pack deploy$ kubectl pack delete [--purge]
$ kubectl pack repo add appscode https://charts.appscode.com/stable/$ kubectl pack repo update$ kubectl pack search repo appscode/vault-operator --version v0.3.0
$ kubectl pack list
$ kubectl pack up$ kubectl pack diff |
| --- |

**Replace Dependency with Bundles**

We believe composition is better than the inheritance style dependency management system currently supported in Helm. We call this a &quot;Bundle&quot; as in a bundle of charts that should be used together but users can free to install the charts they need. This will allow users to install a wordpress chart without installing a postgres chart as a dependency. I intend to write a separate document explaining how this might work.

[1](#sdfootnote1anc)[https://github.com/kubernetes-sigs/application](https://github.com/kubernetes-sigs/application)

[2](#sdfootnote2anc)[https://helm.sh/docs/intro/using\_helm/#helpful-options-for-installupgraderollback](https://helm.sh/docs/intro/using_helm/#helpful-options-for-installupgraderollback)

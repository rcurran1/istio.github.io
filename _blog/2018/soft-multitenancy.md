---
title: "Soft Multitenancy Support"
overview: Using kubernetes namespace and RBAC to create an Istio soft multitenancy environment
publish_date: March 21, 2018
subtitle: Using multiple Istio control planes and RBAC to create multitenancy
attribution: John Joyce and Rich Curran

order: 92

layout: blog
type: markdown
redirect_from: "/blog/soft-multitenancy.html"
---
{% include home.html %}

## Defining soft multitenancy 
Multitenancy is commonly used in many environments across many different applications, but the
implementation details and functionality provided on per tenant basis does not follow one
model in all environments. While discussing the strengths and benefits of deploying
applications on top of Istio it became apparent that there are multitenant use cases for
Istio. A couple different multi-tenant models could be considered.
1.	A single mesh with multiple applications one for each tenant on the mesh. The admin gets
control and visibility mesh wide and across all applications. While the tenant only gets
control of his/her specific application. 
1.	A single Istio control plane with multiple meshes (one per tenant) under that control
plane. The admin gets control and visibility across the entire Istio control plane. While the
tenant only gets control of his/her specific mesh.
1.	A single Kubernetes control plane but multiple Istio control planes one per tenant. The
admin gets control and visibility across all the Istio control planes. While the tenant only
gets control of his/her specific Istio instance. 
1.	A single cloud environment (admin controlled) – but multiple kubernetes control planes
(tenant controlled)

The fourth model doesn’t satisfy most use cases as most administrators prefer
a common kuberenetes control plane with which they provide as a PaaS to their tenants.
Additionally, case 4 is easily provided in many environments already.  
Current Istio capabilities are poorly suited to support the first model as it lacks
sufficient RBAC capabilities to support admin versus tenant operations. Additionally,
having multiple tenants under one mesh is too insecure with the current mesh model. 
The current Istio paradigm assumes a single mesh per Istio control plane. The needed
changes to support this mode are substantial and would also require RBAC changes to the
Istio resources. Although some operators might want to provide Istio as PaaS and have
resource sharing between tenants the benefits of sharing an Istio control plane across
tenants is marginal compared to sharing a kubernetes cluster across tenants.

This blog describes how to provide Option 3. Current Istio capabilities are fairly well
suited to providing option 3 as namespace based scoping is already widely supported in
most Istio modules. Best practices for deploying multiple tenant applications per cluster
require the use of a namespace. This blog will provide a recipe for deploying multiple
tenants on a single Kubernetes cluster while providing a unique Istio control plane for
each tenants use.  

## Deployment details
#### Multiple Istio control planes
Deploying multiple Istio control planes is as simple as replacing all `namespace` references
in a manifest file with the desired namespace. Using istio.yaml as an example, if two tenant
level istio control planes are required, the first can use the istio.yaml default name of
*istio-system* and a second control plane can be created by creating a new yaml file with
a different namespace.
```bash
cat istio.yaml | sed s/istio-system/istio-system1/g > istio-system1.yaml
```
Note that the execution of these two yaml files is the responsiblity of the cluser
administator, not the tenant level adminstator. Additional RBAC restrictions will also
be configured and applied by the cluster administator limiting the tenant admin to only
the assigned namespace.

If the Istio [addons]({{home}}/docs/tasks/telemetry/) are required then the manifests must
be updated to match the configured `namespace` in use by the tenant's Istio control plane.

#### RBAC applied to Istio control planes
To restrict a specific user(s) to a single Istio namespace, the cluster admin would
apply a manifest similar to the one listed below which restricts the user *sales-admin*
to the namespace *istio-system1*.
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: istio-system1 
  name: ns-access-for-sales-admin-istio-system1
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["*"]
  verbs: ["*"]
---
# This role binding allows *sales-admin* to access everything in the *istio-system1* namespace.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: access-all-istio-system1
  namespace: istio-system1
subjects:
- kind: User
  name: sales-admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: ns-access-for-sales-admin-istio-system1
  apiGroup: rbac.authorization.k8s.io

```
#### Watching specific namespaces for service discovery
Update the istio manifest to specify the namespace that pilot should watch for creation of
its xDS cache. This is done by starting the pilot application with the additional command line
arguments,  `--appNamespace, ns-1`.  Where *ns-1* is the namespace that the tenant’s
application will be deployed in. An example snippet from the istio-system1.yaml file is
included below.
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: istio-pilot
  namespace: istio-system1
  annotations:
    sidecar.istio.io/inject: "false"
spec:
  replicas: 1
  template:
    metadata:
      labels:
        istio: pilot
    spec:
      serviceAccountName: istio-pilot-service-account
      containers:
      - name: discovery
        image: docker.io/<user ID>/pilot:<tag>
        imagePullPolicy: IfNotPresent
        args: ["discovery", "-v", "2", "--admission-service", "istio-pilot", "--appNamespace", "ns-1"]
        ports:
        - containerPort: 8080
        - containerPort: 443

```

#### Deploying the tenant application in a namespace
Now that the cluster admin has created the tenant's namspace (ex. *istio-system1*) and
the pilot's service discovery has been configured to watch for a specific application
namespace (ex. *ns-1*), create the application manifests to deploy in that tenant's specific
namespace. Example: 

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ns-1
```
And add the namespace reference to each resource type included in the applications manifest
file.  For example:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: details
  labels:
    app: details
  namespace: ns-1
```
## Issues
* The CA (Certificate Authority) and mixer Istio pod logs using the *istio-system* `namespace`
contained 'info' messages for *istio-system1*. 

## References

* Video on kubernetes multitenancy support, [Multi-Tenancy Support & Security Modeling with RBAC and Namespaces](https://www.youtube.com/watch?v=ahwCkJGItkU), and the [supporting slide deck ](-	https://schd.ws/hosted_files/kccncna17/21/Multi-tenancy%20Support%20%26%20Security%20Modeling%20with%20RBAC%20and%20Namespaces.pdf).
* Kubecon talk on security that discusses kubernetes support for "Cooperative soft mult-tenancy", [Building for Trust: How to Secure Your Kubernetes ](https://www.youtube.com/watch?v=YRR-kZub0cA).
* Kubernetes documentation on [RBAC](https://kubernetes.io/docs/admin/authorization/rbac/) and [namespaces](https://kubernetes.io/docs/tasks/administer-cluster/namespaces-walkthrough/).
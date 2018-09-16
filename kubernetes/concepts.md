## Kubernetes Notes

### Introduction

These notes track from the Concepts section of the [Kubernetes website](https://kubernetes.io/docs/concepts/). I chose certain things that I thought would be interesting to talk about, and didn't include everything. This is designed as a sort of quick gloss and cheat sheet for myself, and other folks who maybe already know a thing or two about what k8s is. Having created these notes in order, I don't recommend reading them in order not know anything about Kubernetes. I would probably in order, read about

1. [Pods](#pods)
2. [The System Architecture](#kubernetes-architecture)
3. [Kubectl](#object-management-using-kubectl)
4. [Configuration](#configuration)

Beyond that, it's sort of pick and choose (and once you get these, you can read straight through as well).

`Terminology:` k8s is a common abbreviation of Kubernetes (`len("ubernete") === 8`), and I use it frequently in these notes

### Working with Kubernetes Objects

- Names: resources have names (which must be unique across resource type) and UIDs
- Namespaces: allows you to re-use names based on context (only for hundreds of users)
- Labels: key value pairs attached to resources.
- Annotations: free verse labels (unrestricted)
- Field Selectors: let you query on things like "status=running"

### Object Management Using kubectl

#### Management techniques
- Imperative commands - operate on live objects. Provide operations to kubectl command.
- Imperative object configuration - specify a file that contains the definition of an object
- Declarative object configuration - process all the configuration files in multiple directories, and CUD live objects based on specifications in the directory.
- You can run migrations between imperative commands and imperative object configuration by exporting the state to a file, and then updating that file.
- Declarative apply could potentially affect all resources:
- What's the best way to manage a directory? (My take: it's a monorepo: https://twitter.com/dnrubenstein/status/1038523801368256515)
- If you update a configuration field that isn't specified in a declarative object configuration (I guess it resolves to whatever the default is) with an imperative command, and then re-apply 
- Deleting resources: run `kubectl delete -f <file_where_a_resouce_is_specified>`. You can also delete the file, maybe? This is not super clear.
- `kubectl apply` does a diff and then sets the diff.
- Default field values: api server sets values to default when not specified in object creation strategy. Recommended that defaults are specified in the file (I like this a lot).

### Compute, Storage, and Networking Extensions

#### Certificates: they can be passed around to client nodes (doesn't seem super interesting for me right now)

#### AWS notes (this is what we use at work)
- EKS uses the private DNS name of the AWS instance as the name of the node object
- EKS supports certain custom annotations on load balancer services (access logs). Can also use annotations to integrate with AWS tags.
- There a couple of other tags here, which determine a lot of things about connections (http/s, certificates, load balancing, etc.).

Snark: Of course IBM is in on the k8s game.

#### Managing Resources
- Organizing resource configurations: you can put multiple things in the same file (separate by `---`).
- Recommended to put resources related to the same microservice or application tier in the same file (I assume this means `{staging,production,test}`), and then group multiple files of the same application in the same folder (e.g., `userapplication/{staging,production}.yaml` (you can also create from a URL)
- Nuke combined sets of resource (a service and a deployment) by deleting by label.
- kubectl supports a bunch of bulk operations (name extracting, deleting resources) (-R flag to do things recursively on directories)
- Example: use track label to manage canary deployments
- There are destructive updates and non-destructive updates here. Consult docs for more detail. Generally labels and annotations are non-destructive updates.

#### Cluster Networking
- k8s has a bunch of communication challenges to solve. This section focuses on pod to pod communications.
- k8s assumes that pods (each with own ip address) can communicate with all other pods. 
- Docker model: only containers on the same physical machine can communicate with one another.
- k8s model: all containers can communicate with other containers without NaT, nodes can communicate with all containers without NaT. Has less friction (goal was to move applications from VMs to containers in k8s), but docker is not just plug and play.
- Pod scope: containers in pod share network namespace (IP address). This means can reach other on localhost, and must coordinate ports.

#### Logging Architecture
- Logs are handled at the pod level, and can be further examined at the container level. Logs are handled by a system container, and need to be rotated sequentially (open q: how do you ship logs elsewhere?)
- System component logs (e.g., logs for kubelet) are stored in files on the cluster nodes.
- k8s does not have a cluster level logging solution (does support Stackdriver Logging and Elasticsearch, and there are also some sidecar container examples)

#### Configuring kubelet Garbage Collection
- Garbage collection: will clean up unused images and containers. 
- There are some user configuration options here, but the defaults are encouraged.

#### Federation
- Seems complicated. Not helpful unless a configuration has multiple clusters

#### Proxies in Kubernetes: only two (of five) that actually matter
	i. kubectl proxy: proxies from a user's desktop or a pod to the k8s apiserver
	ii: apiserver proxy, which is a bastion in the apiserver that connects a user outside of the cluster to clusterips that aren't reachable, can reach a node, pod, or service (only service had)

#### Controller manager metrics
- Provide metric views on controllers, emitted in Prometheus, and available at http://localhost:10252/metrics

#### Installing add-ons
- They exist. You can install them. This is outside of the scope of the notes (I did not read this section beyond the opening paragraph)

### Kubernetes Architecture

#### Nodes
- A node is a worker machine, contains the services to run pods and is managed by the master components. Nodes contain various system services (container runtime, kubelete, and kube-proxy)
- A Node's status consists of addresses, condition (is the node healthy), capacity (resources available), and info (k8s meta-data). 
- Nodes are not inherently managed by k8s, but the node controller manages various aspects of nodes (allowed into the cluster, what ips are assigned). 
- Node controller also handles pod movement across nodes and other things (availability zones). 
- The preferred pattern for node registration is self-registration. 
- Capacity of the node is part of the node object, the scheduler only ensures there are enough resources for all the pods on a node (does not care about system resources or other non-container resources).

#### Master-Node communication
- Communication from the cluster to master terminates at the apiserver, typically port 443 with client authorization (and nodes should have a public certificate so they can securely connect to the apiserver)
- Communication from master to the cluster: either apiserver to kubelet (on pods) or apiserver to nodes, pods, and services (These connections are not safe to run over untrusted networks)

#### Concepts Underlying the Cloud Controller Manager
- This manager was created to allow cloud specific code and k8s to evolve independently. In this model, the manager runs controllers for nodes, routes, and services that are dependent on the cloud manager. There are a lot of other details on how to integrate k8s as a cloud provider.

### Extending Kubernetes

#### Extending your Kubernetes Cluster
- Extensions can be separated into configurations (which are flags, api resources) and extensions (running additional services). This about extensions.
- A typical pattern for writing extensions is the Controller Pattern (read an object's `.spec`, do something, and update the object's `.status`). The other pattern is for k8s to make requests to a remote service (k8s is the client).
- Most layers of the k8s infrastructure can be extended (kubectl, apiserver, resources, scheduler, controllers, networking, and storage).
- Custom resources can be used if you want new controllers or declarative APIs (esp. if managed by kubectl). You cannot change existing API groups. 
- There are also lots of different ways to extend infrastructure (each one has its own link and long list).

#### Extending the Kubernetes API with the aggregation layer
- Aggregation layer allows k8s style APIs. (I do not know what this means). 

#### Custom Resources
- There are reasons to use a custom resource and reasons not to use one (there's a long chart). But in general, custom resources should not encode lots of complicated business logic.
- Custom resources can be managed using the typical k8s API (e.g., `kubectl`).
- Custom resources can be added to your cluster via a CustomResourceDefinition or API server aggregation. `CustomResourceDefinition` enables less customization (but see the previous point). 

#### Network Plugins (This is in alpha as of v1.11)
- You can add plugins to enable fancy things like cross-node networking.

#### Device Plugins
- This are used if a vendor needs to be able to advertise a specific hardware feature to the kubelet on the node (like a GPU) for services and or k8s resources that require that hardware feature.

#### Service Catalog
- This is an extension for applications running in k8s cluster to use managed software offerings (like RDS, CloudSQL, etc). A cluster operator can provision an instance of the managed service, and bind it to make it available to a k8s cluster. 
	- How would you limit the security group access to this?
- The service broker (e.g., AWS) advertises the managed service to the cluster. Then the cluster makes a request to provision an instance of the service. The broker than returns a link to the instance, and gives the cluster connection and other secrets, which are stored in a pod volume. The application can then map the secret into its service. 

### Containers

#### Images
- Images generally behave similarly as in the docker ecosystem. The default image pull policy is to pull if it is not present.
- You can use private registries (AWS using IAM roles)

#### Container Environment Variables
- k8s provides, via system calls or environment variables, information about both the container (e.g., "hostname") and cluster (e.g., the port the service is running on).

#### Container Lifecycle Hooks
- Two hooks are exposed to containers, `PostStart` and `PreStop`, that allow the container to run code at that point in the lifecycle. 
- There are two types of hook handlers: `exec` style commands and http requests against the container itself.
- Hook handlers are designed to be delivered at least once. They should be generally lightweight.

### Pods

#### Overview
- A pod is the smallest unit in the k8s object model. It consists of an application container, storage resources, a network ip, and how that container should run. A common use case is the one pod per container model.
- Multiple co-located and co-managed containers in a single Pod is a relatively advanced use case.
- Each pod is assigned to a unique IP address. Containers inside the pod can communicate with using `localhost`. 
- Pods are rarely managed one at a time. Instead they are managed by controllers, which use a Pod Template to create the pods.

#### Pods (in-depth, I guess)
- Pods as a unit serve to simplify management. They serve as a unit of deployment, scaling, and replication. Everything in a pod lives and dies at the same time (no zombie pods).
- Pods can be used for vertical stacks (LAMP), but in general, pods are not intended to run multiple instances of the same application.
- Pods are not durable entities, and should not be created directly.
- Pods should be allowed to die gracefully, so that requests can complete (use SIGTERM, not SIGKILL). The default grace period during the shutdown of a pod is 30 seconds.

#### Pod Lifecycle
- A pod its status in the `.status.phase` field. There are five options (`pending, running, succeeded, failed, unknown`).
- Containers are probed using a diagnostic performed periodically by the kubelet on a container. They can be either execs, a TCP check, or an HTTP Get request. Probes can be success, failure, or unknown.
- Health checks can either be a `livenessProbe` (should the kubelet keep the container around) or a `readinessProbe` (can the container serve requests).
- There are also additional gates (Pod Readiness gate) that determine access for requests.
- The `restartPolicy` determines whether or not to restart stopped containers applies to all Containers in the Pod (and only those in that Node).
- Pods do not generally disappear until they are destroyed.
- A Pod state is different from a condition (a Pod can have multiple conditions, but only one of the five states).

#### Init Containers
- These are specialized Containers that precede app Containers, and are generally used for specialized configuration tasks. This seems like the CloudFormation equivalent of `dependsOn`.
- Init Containers are executed in order, and each must exit successfully before the next one is run. A Pod that restarts has all of the Init Containers run again. 

#### Pod Preset
- A PodPreset injects information into pods at creation time (secrets, volumes, etc).
- When a Pod is create, the system looks at all of the PodPresets, checks the labels on the Pod created, and merges all the resources defined by a PodPreset into the Pod created. If there is a merging error, there is an event that specifies this, and the pod is created without injected resources. Pod specifications are annotated by PodPresets.
- Pods can be excluded from conditions set by PodPreset.

#### Disruptions
- Disruptions can be voluntary (a controller destroys them) and involuntary (hardware failure). 
- Planning helps: make sure that a Pod has the resources it needs, replicate your application if you need HA, and spread across racks and zones for more HA.
- PodDisruptionBudgets (PDB): limits the number of pods on a replication that are down from voluntary disruptions. PDBs cannot prevent involuntary disruptions, but these do count against the budget.
- PDBs are a useful tool to enable cluster changes, such as node or system upgrades.

### Controllers

#### ReplicaSet
- ensures that a specified number of pod replicas are running at any given time. A ReplicaSet manages all the pods with the labels that match its `.spec.selector` field, regardless of whether or not the ReplicaSet owns the pod in question.
- ReplicaSet is not preferred, instead it is recommended to work directly with Deployments, which manage their own ReplicaSets under the hood.

#### ReplicationController
- Previous generation ReplicaSet

#### Deployments
- Provides updates for Pods and ReplicaSets. Describe a desired state for a Deployment, and the controller will change the present state over to the new state.
- A Deployment is created by running `kubectl create` on a deployment declarative .yaml file. 
- A deployment is updated either by setting directly or by editing the file. The update then triggers a rollout if the update changes the pod template (but not, say, if the deployment scale is changed.
- If a rollout is in-flight, and a new deployment object is picked up by the controller, the items in the previous rollout are immediately considered outdated and subject to be scaled down.
- Changing a deployment's label selectors (e.g., what pods are affected by the rollout) is complicated, and is generally discouraged.
- The entire rollout history of a deployment is kept, and can be rolled back to any previous revision.
- The timing on rollouts is highly configurable, including how long a rollout has, how many new pods can be created at any given time, and how long a stuck rollout can persist.

#### StatefulSets
- A workload to manage deployment and scaling of a set of pods, and maintain the identity of its pods across deployments. 
- Replicas in a StatefulSet are strictly managed in their rate of deployment, and the volumes that they are allowed to keep.

#### DaemonSets
- Ensures that all or some nodes run a copy of a pod. Typically using for things like a storage daemon, a log collection agent, or a node monitoring daemon.
- DaemonSets start before other Pods

#### Garbage Collection
- k8s objects can be owned by other objects (dependents). Garbage collection whether or not objects that do not have an owner are also deleted, and when. 
- This is configurable: dependents can either be deleted before their owner or after their owner. You can also decide to "orphan" an object by not deleting the dependents of its owner. 

#### Jobs
- One or mode pods wherein a specified number successfully terminate. A Job will cleanup the pods it creates.
- Jobs can be non-parallel (only one pod is started unless the pod fails), parallel with a fixed completion count, and parallel with a work queue (coordinate with themselves what each should work on). 
- Jobs have the usual Pod characteristics around handling failures, although parallel job execution makes things a little bit more complicated. 
- When a job completes, pods are no longer created, but pods are not automatically cleaned up. When a job is deleted, all of its pods are deleted.
- Jobs are designed to support parallel processing on work items. Jobs are not designed to support closely communicating processing.
- Jobs have a pretty standard set of tradeoffs around parallelism.

#### CronJob
- Creates a Job on a time-based schedule (all times UTC).
- There are certain edge cases where two jobs might be created, or no job might be created. Jobs should be idempotent. 

### Configuration

#### Configuration Best Practices
- Use configuration files, and group similar things into a single file (YAML preferred) and similar places. `kubectl` can be called on a directory.
- Put object descriptions in annotations. 
- Avoid "naked" pods (e.g., not bound to a Deployment or ReplicaSet).
- Create a Service before its backend workload. If using code that talks to a service, use the DNS name of the service.
- Don't specify a `hostPort` and `hostNetwork` for a pod unless absolutely necessary
- Use labels for semantic attributes of an application or deployment.

#### Manging Compute Resources for Containers
- CPU and memory can both be managed in containers. Capacity checks are used to schedule Pods with resource requests, and the limits are used, not the actual usage.
- v1.11 (beta). Ephemeral storage may also be managed. If a container exceeds its storage limit, then the pod will be evicted. 
- Extended resources (like device plugins, node-level) can also be managed.

#### Assigning Pods to Nodes
- Pods can be assigned to run only on some nodes, or preferred on some nodes. This is all done using label selectors. 
- Affinity (and anti-affinity) can also be used to put pods next to other pods as necessary.
	- The example here does not make a lot of sense: why would you need to use affinity and anti-affinity to put a cache pod on the same node as the webserver pod if you could just put the cache and webserver in the same pod? Maybe you don't want the webserver to die and kill a warm cache?

#### Taints and Tolerations
- Taints (on nodes) allow nodes to repel pods. Tolerations (on pods) allow pods to be schedule on nodes with matching taints.

#### Secrets
- A Secret is an object that contains a small amount of sensitive behavior. To be used, a secret must be referenced by a pod, either with kubelet when pulling images for that pod, or as files in a volume.
- Secrets have lots of complicated associated properties with who can access them and when. Use things like RBAC (role-based access control) to determine what pods and other domain objects can use them. 
- Being able to use a secret is functionally the same as being able to see the secret's value.

#### Organization Cluster Access Using kubeconfig Files
- If you have multiple clusters, you need to manage access to multiple files. 
- `kubectl` uses an environment variable called `KUBECONFIG` that determines what configuration files (and by extension, clusters), a user might want to access. 
- If there are multiple files, and the `--context` flag is not set in a `kubectl` command (which automatically sets the context from that file), the configurations in these files are merged using some complicated rules. The merge algorithm, while interesting, is out of scope here.

#### Pod Priority and Preemption
- Pods can have priority. Priority indicates the importance of a Pod relative to other Pods when making scheduling and eviction decisions.
- In 1.11 and later, a PriorityClass is a non-namespaced that maps a name (string) to a priority (integer, higher is better).
- A PriorityClass can also be set as the global default for any pod without a priority. At most one such class can be set in the system at once. 
- When pods are created, they go in a queue to be scheduled. If that pod cannot be scheduled, preemption kicks in -> if there's a node with lower priority pods, they are gracefully shut down, and then killed.

### Services, Load Balancing and Networking

#### Services
- A k8s services is an abstraction of pods and an access policy. What pods are targeted by a service, is determined by a Label Selector.
- k8s services can either target a single port or multiple ports (multiple ports are disambiguated by string names).
- k8s does service discovery by populating pods with environment variables for each active service, like `{SVCNAME}_SERVICE_HOST` and `{SVCNAME}_SERVICE_PORT`. At creation time, Pods only have the environment variables of service names, new services do not get automatically created. 
- DNS: an optional add-on that is strongly recommended to enable DNS lookup for new services, and creates a set of DNS records for each service. Then all Pods can do name resolution of Services automatically.
- Services can be headless (single IP, no load balancing).
- There are several `ServiceTypes` that make services accessible inside and outside various parts of the cluster.

#### DNS for Services and Pods
- Every service in the cluster is assigned a DNS name, which by default includes the Pod's namespace and cluster's default domain -> service `foo` in namespace `bar` is available at `foo.bar`.
- Both normal and "headless" services are assigned an A record at `my-svc.my-namespace.svc.cluster.local`, but the record for normal services resolves to the cluster IP, while the headless service gets a set of IP address (clients are expected to be able to deal with this). 
- Pods get an A record: `pod-ip-address.my-namespace.pod.cluster.local`. The Pod's host name is the `metadata.name` field.
- Multiple DNS resolution policies are available, and can be set on a Pod first basis. The default is `ClusterFirst`, wherein any DNS request that doesn't match the configured cluster domain suffix, gets forwarded to the upstream nameserver.

#### Connecting Applications with Services
- Services can be directly created from Deployment with the `expose` command, which creates a service from exactly all the pods in that Deployment.
- Services can be exposed to an external IP address, either with NodePorts or LoadBalancers.

#### Ingress
- An Ingress is a collection of rules that allow inbound connections to reach the cluster services. 
- To enable ingress, the cluster must have an Ingress controller, which is a unique controller, and is not enabled by default when the cluster is created.
- Single service ingress: creates a default backend for all access to the cluster.
- Simple fanout: does path based routing into the cluster.
- Name based virtual hosting: routes based on the Host header.

#### Network Policies:
- A NetworkPolicy determines what traffic pods can accept. Pods are non-isolated by default, and can become isolated by having a `NetworkPolicy` that selects them.
- A NetworkPolicy can restrict both ingress and egress traffic, as determined by IP blocks (CIDR), pod labels, and namespaces.
- One can also create a collection of network rules to default deny or allow all ingress and egress traffic to pods.

#### Adding entires to Pod /etc/hosts/ with HostAliases
- This can be done to manage the host file. Kubelet manages the `hosts` file for each container of the pod to prevent Docker from changing the file after containers are started.

### Storage

#### Volumes
- k8s supports volumes to enable sharing of files between containers in a Pod, and the saving of files across the restart and starting of Pods. 
- I am not sure about this, but Volumes cannot be shared across Nodes (this makes sense if you think about it for a bit, but is a tad frustrating).
- k8s supports many types of Volumes, including most major cloud providers, and a bunch of others.
- Volumes can be split with subPaths to share a volume for different uses inside a single pod.

#### Persistent Volumes
- Mount propagation enables sharing volumes between other containers on the same pod, and even on different pods across the same node.
- Persistent volumes can be claimed and allocated like other available compute resources (think of this as the storage equivalent of node allocation of CPU and memory), with top level objects `PersistentVolume` and `PersistentVolumeClaim`.
- A pod can use a `PersistentVolumeClaim` as a volume instead.

#### StorageClass
- A StorageClass provides a way to describe the types of storage they offer (think of this as the equivalent of requesting access to GPU devices in compute).
- A StorageClass using parameters to describe the volumes that belong to the class. They are dependent on the configuration of the storage provisioner (e.g., AWS is different from GCE is different from glusterfs).

#### DynamicVolumeProvisioning
- Allows storage volumes to be created on-demand from a precreated StorageClass. A cluster can also have cluster-wide defaults for storage claims, which are defined in a PersistentVolumeClaim.

#### Node-specific Volume Limits
- There is a finite number of volumes that can attach to a node at one time.
- Volumes can be dynamically limited based on the type of node.

### Policies

#### Resource Quotas
- Resource Quotas enable administrators to use more than its share of resources, across compute, storage, and objects.
- Quotas can have scopes, which limit the quotas to the addressed resources that match the descriptions defined in the scope.
- (v1.11 alpha). Quotas can be set per PriorityClass.
- Quotas can be attached to either the requested amount of a compute resource or the limit of a compute resource.
- Quotas are absolute, and not based on the total resources available for allocation.

#### Pod Security Policies
- Defined at the system level to only allow pods that meet the listed security requirements into the cluster.
- The pods must be able to view these policies, so there's some complicated stuff about RBAC here.



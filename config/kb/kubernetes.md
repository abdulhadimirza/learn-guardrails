# Kubernetes Architecture & Concepts: RAG-Optimized Knowledge Base

## Kubernetes Architecture Overview

Kubernetes is a portable, extensible, open-source platform for managing containerized workloads and services. It facilitates both declarative configuration and automation.

### Control Plane Components

The control plane's components make global decisions about the cluster (for example, scheduling), as well as detecting and responding to cluster events (for example, starting up a new pod when a deployment's `replicas` field is unsatisfied).

#### kube-apiserver

The API server is a component of the Kubernetes control plane that exposes the Kubernetes API. The API server is the front end for the Kubernetes control plane. It is designed to scale horizontally—that is, it scales by deploying more instances. You can run several instances of `kube-apiserver` and balance traffic among those instances.

#### etcd

Consistent and highly-available key-value store used as Kubernetes' backing store for all cluster data. Always have a backup plan for etcd data for your Kubernetes clusters.

#### kube-scheduler

Control plane component that watches for newly created Pods with no assigned node, and selects a node for them to run on. Factors taken into account for scheduling decisions include: individual and collective resource requirements, hardware/software/policy constraints, affinity and anti-affinity specifications, data locality, inter-workload interference, and deadlines.

#### kube-controller-manager

Control plane component that runs controller processes. Logically, each controller is a separate process, but to reduce complexity, they are all compiled into a single binary and run in a single process.
Some types of these controllers are:

- **Node controller**: Responsible for noticing and responding when nodes go down.
- **Job controller**: Watches for Job objects that represent one-off tasks, then creates Pods to run those tasks to completion.
- **EndpointSlice controller**: Populates EndpointSlice objects (to provide a link between Services and Pods).
- **ServiceAccount controller**: Create default ServiceAccounts for new namespaces.

### Node Components

Node components run on every node, maintaining running pods and providing the Kubernetes runtime environment.

#### kubelet

An agent that runs on each node in the cluster. It makes sure that containers are running in a Pod. The kubelet takes a set of PodSpecs that are provided through various mechanisms and ensures that the containers described in those PodSpecs are running and healthy. The kubelet doesn't manage containers which were not created by Kubernetes.

#### kube-proxy

`kube-proxy` is a network proxy that runs on each node in your cluster, implementing part of the Kubernetes Service concept. It maintains network rules on nodes. These network rules allow network communication to your Pods from network sessions inside or outside of your cluster.

#### Container Runtime

The container runtime is the software that is responsible for running containers. Kubernetes supports container runtimes such as containerd, CRI-O, and any other implementation of the Kubernetes CRI (Container Runtime Interface).

---

## Core Objects & Workloads

### Pods

Pods are the smallest deployable units of computing that you can create and manage in Kubernetes. A Pod is a group of one or more containers, with shared storage and network resources, and a specification for how to run the containers.

#### Pod Lifecycle Statuses

| Status      | Description                                                                                                                                                                                                                                                        |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `Pending`   | The Pod has been accepted by the Kubernetes cluster, but one or more of the containers has not been set up and made ready to run. This includes time a Pod spends waiting to be scheduled as well as the time spent downloading container images over the network. |
| `Running`   | The Pod has been bound to a node, and all of the containers have been created. At least one container is currently running, or is in the process of starting or restarting.                                                                                        |
| `Succeeded` | All containers in the Pod have terminated in success, and will not be restarted.                                                                                                                                                                                   |
| `Failed`    | All containers in the Pod have terminated, and at least one container has terminated in failure. That is, the container either exited with non-zero status or was terminated by the system.                                                                        |
| `Unknown`   | For some reason the state of the Pod cannot be obtained. This phase typically occurs due to a communication failure with the host where the Pod should be running.                                                                                                 |

### Workload Resources

#### Deployments

A _Deployment_ provides declarative updates for Pods and ReplicaSets. You describe a desired state in a Deployment, and the Deployment Controller changes the actual state to the desired state at a controlled rate.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
  spec:
    containers:
      - name: nginx
        image: nginx:1.25.4
        ports:
          - containerPort: 80
```

#### StatefulSets

`StatefulSet` is the workload API object used to manage stateful applications. Manages the deployment and scaling of a set of Pods, and provides guarantees about the ordering and uniqueness of these Pods. Like a Deployment, a StatefulSet manages Pods that are based on an identical container spec. Unlike a Deployment, a StatefulSet maintains a sticky identity for each of their Pods. These pods are created from the same spec, but are not interchangeable: each has a persistent identifier that it maintains across any rescheduling.

#### DaemonSets

A _DaemonSet_ ensures that all (or some) Nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to them. As nodes are removed from the cluster, those Pods are garbage collected. Deleting a DaemonSet will clean up the Pods it created.

Typical use cases for DaemonSets include:

- Running a cluster storage daemon on every node.
- Running a logs collection daemon on every node, such as Fluentd or Logstash.
- Running a node monitoring daemon on every node, such as Prometheus Node Exporter.

#### Jobs and CronJobs

A **Job** creates one or more Pods and will continue to retry execution of the Pods until a specified number of them successfully terminate. As pods successfully complete, the Job tracks the successful completions.
A **CronJob** manages Jobs run on a time-based schedule. One CronJob object is like one line of a _crontab_ (cron table) file. It runs a Job periodically on a given schedule, written in Cron format.

---

## Services, Load Balancing, and Networking

### Kubernetes Networking Model

Every `Pod` in a cluster gets its own unique cluster-wide IP address. This means you do not need to explicitly create links between `Pods` and you almost never need to deal with mapping container ports to host ports. This creates a clean, backwards-compatible model where `Pods` can be treated much like VMs or physical hosts from the perspective of port allocation, naming, service discovery, load balancing, application configuration, and migration.

### Service Types

Kubernetes `Service` types allow you to specify what kind of Service you want:

| Service Type   | Description                                                                                                                                                                                                                                                                        |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ClusterIP`    | Exposes the Service on a cluster-internal IP. Choosing this value makes the Service only reachable from within the cluster. This is the default `ServiceType`.                                                                                                                     |
| `NodePort`     | Exposes the Service on each Node's IP at a static port (the `NodePort`). A `ClusterIP` Service, to which the `NodePort` Service routes, is automatically created. You'll be able to contact the `NodePort` Service, from outside the cluster, by requesting `<NodeIP>:<NodePort>`. |
| `LoadBalancer` | Exposes the Service externally using a cloud provider's load balancer. `NodePort` and `ClusterIP` Services, to which the external load balancer routes, are automatically created.                                                                                                 |
| `ExternalName` | Maps the Service to the contents of the `externalName` field (e.g. `foo.bar.example.com`), by returning a `CNAME` record with its value. No proxying of any kind is set up.                                                                                                        |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  type: ClusterIP
```

### Ingress

An API object that manages external access to the services in a cluster, typically HTTP. Ingress may provide load balancing, SSL termination and name-based virtual hosting.

#### Ingress Path Types

Each path in an Ingress spec must have a corresponding path type. There are three supported path types:

- **ImplementationSpecific**: With this path type, matching is up to the IngressClass. Implementations can treat this as a separate pathType or treat it identically to Prefix or Exact path types.
- **Exact**: Matches the URL path exactly and with case sensitivity.
- **Prefix**: Matches based on a URL path prefix split by `/`. Matching is case sensitive and done on a path element by element basis.

#### Ingress Path Matching Examples

| Path Type | Configured Path | Request Path | Matches?                     |
| --------- | --------------- | ------------ | ---------------------------- |
| Exact     | `/foo`          | `/foo`       | Yes                          |
| Exact     | `/foo`          | `/bar`       | No                           |
| Exact     | `/foo`          | `/foo/`      | No                           |
| Prefix    | `/foo`          | `/foo/bar`   | Yes                          |
| Prefix    | `/foo/`         | `/foo`       | Yes (ignores trailing slash) |
| Prefix    | `/aaa/bb`       | `/aaa/bbb`   | No (does not match element)  |

---

## Storage

### Persistent Volumes (PV) and Claims (PVC)

- **PersistentVolume (PV)**: A piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using Storage Classes. It is a resource in the cluster just as a node is a cluster resource. PVs are volume plugins like Volumes, but have a lifecycle independent of any individual Pod that uses the PV.
- **PersistentVolumeClaim (PVC)**: A request for storage by a user. It is similar to a Pod. Pods consume node resources and PVCs consume PV resources. Pods can request specific levels of resources (CPU and Memory). Claims can request specific size and access modes (e.g., they can be mounted ReadWriteOnce, ReadOnlyMany or ReadWriteMany).

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  storageClassName: standard
```

### Storage Classes

A `StorageClass` provides a way for administrators to describe the "classes" of storage they offer. Different classes might map to quality-of-service levels, or to backup policies, or to arbitrary policies determined by the cluster administrators. Kubernetes itself is unopinionated about what classes represent. This concept is sometimes called "profiles" in other storage systems.

---

## Configuration & Security

### ConfigMaps and Secrets

- **ConfigMap**: An API object used to store non-confidential data in key-value pairs. Pods can consume ConfigMaps as environment variables, command-line arguments, or as configuration files in a volume. A ConfigMap allows you to decouple environment-specific configuration from your container images, so that your applications are easily portable.
- **Secret**: An object that contains a small amount of sensitive data such as a password, a token, or a key. Using a Secret means you don't need to include confidential data in your application code or container configurations.

### Namespaces

Kubernetes supports multiple virtual clusters backed by the same physical cluster. These virtual clusters are called _Namespaces_. Namespaces provide a scope for names. Names of resources need to be unique within a namespace, but not across namespaces. Namespaces cannot be nested within one another and each Kubernetes resource can only be in one namespace.

---

## Cluster Management CLI Operations

Common interactions with cluster objects using `kubectl` are summarized below:

```bash
# Apply a configuration file to a resource
kubectl apply -f deployment.yaml

# Get status information about running pods
kubectl get pods -n default

# Describe a specific node for capacity and events
kubectl describe node worker-node-1

# View logs from a specific container in a pod
kubectl logs nginx-deployment-76bf49f89-abc12 -c nginx

```

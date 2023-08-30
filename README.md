This demo is using:

1. Management cluster. Both clusterAPI and Sveltos are deployed in the management cluster.
2. A ClusterAPI powered cluster using docker as infrastructure provider.
3. A GKE cluster registered with sveltos.

## Create management cluster

Sveltos provides a quickstart that takes care of point #1 and #2. It does require docker.

Clone [addon-controller](https://github.com/projectsveltos/addon-controller) repo. It has a __quickstart__ makefile target. Simply run __make quickstart__.

When done, you will have a management cluster with ClusterAPI components, Sveltos and a managed cluster powered by ClusterAPI using docker as infrastructure provider.

Otherwise you can install Sveltos into the cluster you want to use as management cluster:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectsveltos/sveltos/main/manifest/manifest.yaml
kubectl apply -f https://raw.githubusercontent.com/projectsveltos/sveltos/main/manifest/default-classifier.yaml
```

ClusterAPI installation process can be found on [ClusterAPI documentation](https://cluster-api.sigs.k8s.io/user/quick-start).

## Register GKE cluster with Sveltos

For this demo I did create a GKE and then simply registered with Sveltos.

Sveltos [documentation](https://projectsveltos.github.io/sveltos/register-cluster/) contains instructions for that.

It consists in:

1. Generating a Kubeconfig to access the GKE cluster with permissions for Sveltos to deploy necessary add-ons;
2. Using [sveltosctl](https://github.com/projectsveltos/sveltosctl) 
```bin/sveltosctl register cluster --cluster=cluster1 --namespace=gke --kubeconfig=<GKE KUBECONFIG>```

## Deploy add-ons

In this demo both clusterAPI Cluster instance and the SveltosCluster instance have label: "env:fv"

![Sveltos add-on deployment](https://github.com/projectsveltos/sveltos/blob/main/docs/assets/addons_deployment.gif)

1. Create a ConfigMap in the management cluster containing a Kyverno ClusterPolicy preventing 
```
wget https://raw.githubusercontent.com/gianlucam76/sveltos_workshop/main/addon_distribution/kyverno_disallow_latest.yaml
kubectl create configmap kyverno-latest --from-file kyverno_disallow_latest.yaml
```
2. Deploy ClusterProfile that deploys Kyverno Helm chart and the Kyverno ClusterPolicy
```
kubectl apply -f https://raw.githubusercontent.com/gianlucam76/sveltos_workshop/main/addon_distribution/clusterprofile_with_kyverno.yaml
```

Sveltos will deploy Kyverno Helm chart and Kyverno ClusterPolicy in all clusters matching __clusterSelector: env=fv__

To see deployed resources in each cluster

```bash
kubectl exec -it -n projectsveltos sveltosctl-0 -- ./sveltosctl show addons
+-----------------------------+--------------------------+-----------+---------------------+---------+-------------------------------+------------------+
|           CLUSTER           |      RESOURCE TYPE       | NAMESPACE |        NAME         | VERSION |             TIME              | CLUSTER PROFILES |
+-----------------------------+--------------------------+-----------+---------------------+---------+-------------------------------+------------------+
| default/clusterapi-workload | helm chart               | kyverno   | kyverno-latest      | 3.0.1   | 2023-08-30 04:41:42 -0700 PDT | kyverno          |
| default/clusterapi-workload | kyverno.io:ClusterPolicy |           | disallow-latest-tag | N/A     | 2023-08-30 04:42:00 -0700 PDT | kyverno          |
| gke/cluster1                | helm chart               | kyverno   | kyverno-latest      | 3.0.1   | 2023-08-30 04:46:55 -0700 PDT | kyverno          |
| gke/cluster1                | kyverno.io:ClusterPolicy |           | disallow-latest-tag | N/A     | 2023-08-30 04:47:33 -0700 PDT | kyverno          |
+-----------------------------+--------------------------+-----------+---------------------+---------+-------------------------------+------------------+
```

## Deploy add-ons based on cluster Kubernetes version

![Sveltos add-on deployment](https://github.com/projectsveltos/sveltos/blob/main/docs/assets/classifier.gif)

Sveltos can classifiy clusters based on their run-time state and deploy add-ons based on that information.

- The GKE cluster used in this demo is running Kubernetes version v1.27.2
- The ClusterAPI powered cluster is running Kubernetes version v1.27.0

We want to ask Sveltos to deploy Nginx Helm chart only if cluster is running a Kubernetes version equal or higher than __v1.27.2__.

We first post a Classifier instance:

```
kubectl apply -f https://raw.githubusercontent.com/gianlucam76/sveltos_workshop/main/cluster_classification/classifier.yaml
```

Such instance asks Sveltos to add label 
```
  - key: version
    value: v1-27-2
```

to any cluster running a Kubernetes version higher or equal to v1.27.2.

Then we post this ClusterProfile which deployes Nginx Helm chart in any cluster matching __ clusterSelector: version=v1-27-2__

```
kubectl apply -f https://raw.githubusercontent.com/gianlucam76/sveltos_workshop/main/cluster_classification/clusterprofile.yaml
```

Using sveltosctl again we can verify deployed add-ons:

```bash
kubectl exec -it -n projectsveltos sveltosctl-0 -- ./sveltosctl show addons
+-----------------------------+--------------------------+-----------+---------------------+---------+-------------------------------+------------------+
|           CLUSTER           |      RESOURCE TYPE       | NAMESPACE |        NAME         | VERSION |             TIME              | CLUSTER PROFILES |
+-----------------------------+--------------------------+-----------+---------------------+---------+-------------------------------+------------------+
| default/clusterapi-workload | helm chart               | kyverno   | kyverno-latest      | 3.0.1   | 2023-08-30 04:41:42 -0700 PDT | kyverno          |
| default/clusterapi-workload | kyverno.io:ClusterPolicy |           | disallow-latest-tag | N/A     | 2023-08-30 04:42:00 -0700 PDT | kyverno          |
| gke/cluster1                | helm chart               | nginx     | ngix-latest         | 0.18.0  | 2023-08-30 04:54:59 -0700 PDT | nginx            |
| gke/cluster1                | helm chart               | kyverno   | kyverno-latest      | 3.0.1   | 2023-08-30 04:46:55 -0700 PDT | kyverno          |
| gke/cluster1                | kyverno.io:ClusterPolicy |           | disallow-latest-tag | N/A     | 2023-08-30 04:47:33 -0700 PDT | kyverno          |
+-----------------------------+--------------------------+-----------+---------------------+---------+-------------------------------+------------------+
```

As expected, Nginx Helm chart has been deployed by Sveltos only in the GKE cluster (only cluster running a Kubernetes version higher or equal to v1.27.2).

## Add-on Deployment Order

When deploying Kubernetes resources in a cluster, it is sometimes necessary to deploy them in a specific order. For example, a CustomResourceDefinition (CRD) must exist before any custom resources of that type can be created.

Sveltos can help you solve this problem by allowing you to specify the order in which Kubernetes resources are deployed.

ClusterProfile order

There are two ways to do this:

- Using the helmCharts field in a ClusterProfile: The helmCharts field allows you to specify a list of Helm charts that need to be deployed. Sveltos will deploy the Helm charts in the order that they are listed in this field.
- Using the policyRefs field in a ClusterProfile: The policyRefs field allows you to reference a list of ConfigMap and Secret resources whose contents need to be deployed. Sveltos will deploy the resources in the order that they are listed in this field.

Here for instance Sveltos will first deploy Prometheus Helm chart only after it will deploy the Grafana Helm chart.

```
kubectl apply -f https://raw.githubusercontent.com/gianlucam76/sveltos_workshop/main/events/helm_order.yaml
```

![Sveltos: Resource Deployment Order](https://github.com/projectsveltos/sveltos/blob/main/docs/assets/helm_chart_order.gif)

Some other time it is necessary to wait for resources to be in a certain state before other resources are deployed.

![Sveltos: Resource Deployment Order](https://github.com/projectsveltos/sveltos/raw/main/docs/assets/sveltos_resource_order.gif)

This prepares ConfigMap containing:
1. PostgreSQL deployment and service
2. A Job that creates a table in the DB
3. Todo app deployment and service
4. A Job that adds an entry in the DB via todo-app

```
wget https://raw.githubusercontent.com/projectsveltos/sveltos/main/docs/assets/postgresql_deployment.yaml
wget https://raw.githubusercontent.com/projectsveltos/sveltos/main/docs/assets/postgresql_service.yaml
kubectl create configmap postgresql-deployment --from-file postgresql_deployment.yaml 
kubectl create configmap postgresql-service --from-file postgresql_service.yaml 
wget https://raw.githubusercontent.com/projectsveltos/sveltos/main/docs/assets/postgresql_job.yaml
kubectl create configmap postgresql-job --from-file postgresql_job.yaml
wget https://raw.githubusercontent.com/projectsveltos/sveltos/main/docs/assets/todo_app.yaml
kubectl create configmap todo-app --from-file todo_app.yaml
wget https://raw.githubusercontent.com/projectsveltos/sveltos/main/docs/assets/todo_insert.yaml
kubectl create configmap todo-insert-data --from-file todo_insert.yaml
```

Then we ask Sveltos to deploy those resources in order

```
kubectl apply -f https://raw.githubusercontent.com/gianlucam76/sveltos_workshop/main/events/deploy_todo_app.yaml
kubectl apply -f https://raw.githubusercontent.com/gianlucam76/sveltos_workshop/main/events/create_table_job.yaml
kubectl apply -f https://raw.githubusercontent.com/gianlucam76/sveltos_workshop/main/events/add_entry.yaml
kubectl apply -f https://raw.githubusercontent.com/gianlucam76/sveltos_workshop/main/events/postgresql.yaml
```

```bash
kubectl exec -it -n projectsveltos sveltosctl-0 -- ./sveltosctl show addons --namespace=default |grep todo
| default/clusterapi-workload | batch:Job                 | todo       | todo-table          | N/A     | 2023-08-30 05:09:00 -0700 PDT | sveltos-qxkbueopqyvp5u3kdld7 |
| default/clusterapi-workload | :Service                  | todo       | todo-gitops         | N/A     | 2023-08-30 05:09:42 -0700 PDT | sveltos-mot1d4nk7s99dkys2u3n |
| default/clusterapi-workload | networking.k8s.io:Ingress | todo       | todo                | N/A     | 2023-08-30 05:09:42 -0700 PDT | sveltos-mot1d4nk7s99dkys2u3n |
| default/clusterapi-workload | batch:Job                 | todo       | todo-insert         | N/A     | 2023-08-30 05:10:25 -0700 PDT | sveltos-0s8azt6e3g00aeltkpmj |
| default/clusterapi-workload | apps:Deployment           | todo       | todo-gitops         | N/A     | 2023-08-30 05:09:42 -0700 PDT | sveltos-mot1d4nk7s99dkys2u3n |
| default/clusterapi-workload | apps:Deployment           | todo       | postgresql          | N/A     | 2023-08-30 05:08:48 -0700 PDT | postgresql                   |
| default/clusterapi-workload | :Service                  | todo       | postgresql          | N/A     | 2023-08-30 05:08:48 -0700 PDT | postgresql                   |
| default/clusterapi-workload | :ServiceAccount           | todo       | todo-gitops         | N/A     | 2023-08-30 05:09:42 -0700 PDT | sveltos-mot1d4nk7s99dkys2u3n |
```

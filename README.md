# Tools
* OpenShift GitOps 1.9.0
* Openshift 4.13
* Gogs

# Installation

## Ansible

Operator installation:

```bash
oc apply -f operator/openshift-gitops.yaml
```

Retrieve ArgoCD route: 

```bash
oc get route -A | grep openshift-gitops-server | awk '{print $3}'
```

Get the ArgoCD admin password: 

```bash
oc -n openshift-gitops get secret openshift-gitops-cluster -o json | jq -r '.data["admin.password"]' | base64 -d
```

ArgoCD needs some privileges to create specific resources. In this demo, we'll apply cluster-role to ArgoCD to avoid the fine-grain RBAC.

```bash
oc apply -f gitops/cluster-role.yaml
```

Now, we apply the bootstrap application:

```bash
oc apply -f gitops/argocd-app-bootstrap.yaml
```

The bootstrap application initializes the cluster with the necessary dependencies and configuration. This repository is structure functionally:

* **apps**: this folder contains the configuration of the applications that are running on the cluster.
* **dependencies**: our system needs some dependencies which are installed by ArgoCD.
* **gitops**: contains all the ArgoCD objects and cluster configuration as security role bindings.
* **operator**: operators definitions
* **pipelines**: pipelines definitions which are managed by ArgoCD too.
* **tasks**: our own tasks definitions.
* **triggers**: triggers to configure and deploy automatically the pipelines.
* **workspaces**: pvc creation. You also can use a PVC template. 

## Continous Deployment with ArgoCD (CD)

Once, our application image is in the containers' registry, we can deploy it. 

ArgoCD ensures that the definition of our deployment is exactly what we have in a Git repository. 

In this example, our pipeline finishes changing the image version in a git repository that ArgoCD is watching. When ArgoCD detects the change, it updates the OpenShift deployment. 

# Run the demo

If you've completed the installation section, you can start this step. 

At this moment, we've deployed all our dependencies, security and ArgoCD applications but we need our code. I've implemented two repositories, one for a Quarkus application and another with a Helm chart that contains the deployment definition. 

* [Quarkus application](https://github.com/dbgjerez/workshop-tekton-argocd-app-quarkus)
* [Helm chart](https://github.com/dbgjerez/workshop-tekton-argocd-app-quarkus-config)

Now, it's the moment to play with them.

## Gogs

Although our repositories are in GitHub we fork them in an internal Git server to avoid a lot of unnecessary commits. 

This internal server is Gogs. You can enter in ArgoCD and visualize the state of Gogs, if it is correctly synchronized we can enter with user and password ```gogs```.

```bash
oc get route -n gogs gogs -o template --template='{{"http://"}}{{.spec.host}}'
```

We can see that we don't have repositories, so we'll initialize it with a specific pipeline. To run it: 

```bash
oc apply -f gitops/init-pipeline-run.yaml
```

We can enter in OpenShift console and see the pipeline execution in the `Pipeline```` tab:

![app pipeline](images/init-pipeline.png)

After some seconds, we can see the projects in gogs.

![Gogs repositories](images/gogs-apps.png)

## Change the application code

At this point, we can clone and modify some code in the application. When we push the changes to the repository our application pipeline will start. We can see it in the same place as the gogs configuration:

![app pipeline](images/quarkus-pipeline.png)

## Check it in ArgoCD

In ArgoCD we have three applications, one per environment (dev, pre and prod), so we can inspect each and test directly the application.

![app pipeline](images/quarkus-apps.png)
# Kubernetes GitOps Flux Repo

This is an example on how I would structure a 1:1 (repo-to-single cluster)
git repo for a Kubernetes cluster. [Kubernetes with Argo](https://github.com/christianh814/example-kubernetes-go-repo)
repo I created.

This example assumes (as I mentioned in the 1:1 part above) that it's a
single repo for a single cluster. However, this can be modified (quite
easily) for poly/mono repos or for multiple clusters. This is meant as
a good starting point and not what your final repo will look like.

This repo is based on Flux, but uses the same(-ish) repo structure as
my Argo CD example.

# Structure

Below is an explanation on how this repo is laid out. You'll notice
that I use [Kustomize](https://kustomize.io/) heavily. I do this since I
follow the [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)
principal when it comes to YAML files.

```shell
cluster-XXXX/ # 1
├── bootstrap # 2
│   ├── base
│   │   └── kustomization.yaml
│   └── overlays
│       └── default
│           └── kustomization.yaml
├── components # 3
│   ├── gitrepositories
│   │   ├── cluster-gitrepo.yaml
│   │   └── kustomization.yaml
│   ├── helmreleases
│   │   ├── kustomization.yaml
│   │   └── sample-quarkus.yaml
│   ├── helmrepositories
│   │   ├── kustomization.yaml
│   │   └── redhat-helm-charts.yaml
│   └── kustomizations
│       ├── core-components-kustomization.yaml
│       ├── kustomization.yaml
│       └── tenants-kustomization.yaml
├── core # 4
│   ├── gitops-controller
│   │   └── kustomization.yaml
│   └── sample-admin-workload
│       ├── kustomization.yaml
│       └── sample-admin-config.yaml
└── tenants # 5
    ├── bgd-blue
    │   ├── bgd-deployment.yaml
    │   └── kustomization.yaml
    └── myapp
        ├── kustomization.yaml
        ├── myapp-deployment.yaml
        ├── myapp-ns.yaml
        └── myapp-service.yaml
```

|#|Directory Name|Description|
|---|----------------|-----------------|
| 1. |`cluster-XXXX`| This is the cluster name. This name should be unique to the specific cluster you're targeting. If you're using CAPI, this should be the name of your cluster, the output of `kubectl get cluster`|
| 2. | `bootstrap` | This is where bootstrapping specifc configurations are stored. These are items that get the cluster/automation started. They are usually install manifests.<br /><br />`base` is where are the "common" YAML would live and `overlays` are configurations specific to the cluster.<br /><br />The `kustomization.yaml` file in `default` has `cluster-XXXX/components/gitrepositories/`, `cluster-XXXX/components/helmreleases`, `cluster-XXXX/components/helmrepositories` and `cluster-XXXX/components/kustomizations` as a part of it's `bases` config.|
| 3. | `components` | This is where specific components for the GitOps Controller lives (in this case Flux) .<br /><br />`gitrepositories` is where all the configuration YAMLs to access this (and other) git repos live, `helmrepositories` is where the configuration for your helm repositories YAMLs live, `helmreleases` is where your HelmReleases live, and `kustomizations` are where Kustomizations for this cluster live.<br /><br />Other things that can live here are [other Flux components](https://fluxcd.io/docs/components/) (each in their repsective directories).|
| 4. | `core` | This is where YAML for the core functionality of the cluster live. Here is where the Kubernetes administrator will put things that is necissary for the functionality of the cluster (like cluster configs or cluster workloads).<br /><br />Under `gitops-controller` is where you are using Flux to manage itself. The `kustomization.yaml` file uses `cluster-XXXX/bootstrap/overlays/default` in it's `bases` configuration. Everything in this `core` directory gets deployed as a part of the `kustomization` found under `cluster-XXXX/components/kustomizations/core-components-kustomization.yaml`.<br /><br />To add a new "core functionality" workoad, one needs to add a directory with some yaml in the `core` directory. See the `sample-admin-config` directory as an example.|
| 5. | `tenants` | This is where the workloads for this cluster live.<br /><br />Similar to `core`, the `tenants` directory gets loaded as part of a `kustomization` that is under `cluster-XXXX/components/kustomizations/tenants-kustomization.yaml`.<br /><br />This is where Devlopers/Release Engineers do the work. They just need to commit a directory with some YAML and the kustomization takes care of creating the workload.<br /><br /> **Note** that `bgd-blue/kustomization.yaml` file points to another Git repo. This is to show that you can host your YAML in one repo, or many repos.|

# Testing

To see this in action, first get yourself a cluster (using [kind](kind.sigs.k8s.io/) as an example)

```shell
kind create cluster
```

Then, just apply this repo.

```shell
until kubectl apply -k https://github.com/christianh814/example-kubernetes-goflux-repo/cluster-XXXX/bootstrap/overlays/default; do sleep 3; done
```

This should give you 2 `kustomizations`

```shell
$ kubectl get kustomizations -n flux-system 
NAME          READY   STATUS                                                            AGE
flux-system   True    Applied revision: main/111881dc371dd1b2abd823450f01ee085aff3726   3m50s
tenants       True    Applied revision: main/111881dc371dd1b2abd823450f01ee085aff3726   3m50s
```

This should also load your git source

```shell
$ kubectl get gitrepositories -n flux-system
NAME          URL                                                               READY   STATUS                                                            AGE
flux-system   https://github.com/christianh814/example-kubernetes-goflux-repo   True    Fetched revision: main/111881dc371dd1b2abd823450f01ee085aff3726   4m25s
```


# Enjoy

Fork and Enjoy!

> :warning: Feel free to fork this and play around with it, but remember if you'll maybe have to change the where the bases are pointing to if you're changing names of things.

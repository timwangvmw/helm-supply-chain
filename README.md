# TAP Supply Chain to deploy a helm chart using FluxCD

## Why do this

Many organizations already have helm charts developed that are fine tuned to their applications needs.
While moving to the TAP native, knative deployment mechanism is very appealing to many, not all solutions are solved in this manner. Having a way to deploy a helm chart via TAP, while still enabling the rest of the benefits of the platform (security, testing, visibility, abstraction, etc.) is a key part in TAP gaining adoption.

## Requirments

* this has only been tested in internet accessible environments. in Air Gapped deployments, your mileage may vary
* this has only been tested with the GitOps supply chain mechanism. With RegistryOps, your mileage may vary

## How to do this

### Overview

TAP already includes a key component for this which is The FluxCD Source Controller.
With the source controller, we have access to 2 CRDs that interest us:

1. HelmRepository
2. HelmChart

While these 2 resources are great, we need one other CRD which is in charge of the actual deployment and reocncilliation of the helm releases, which is called HelmRelease.

This CRD is provided as part of the FluxCD Helm Controller.

### Installing FluxCD Helm Controller

FluxCD Helm Controller can be easily installed from the Tanzu Community Edition package via the following commands:

```bash
docker load -i fluxcd-helm-controller/images/imgpkgBundle.tar
docker tag dbf594e50e25 {mega-harbor-url}/{project-name}/fluxcd-helm-controller-bundle:0.17.2
docker push {harbor-url}/{project-name}/fluxcd-helm-controller-bundle:0.17.2
imgpkg pull -b {mega-harbor-url}/{project-name}/fluxcd-helm-controller-bundle:0.17.2 -o fluxcd-helm-controller/controller-yaml
vi fluxcd-helm-controller/controller-yaml/.imgpkg/images.yml  # modidy image to the real one !!!!! with digest form not tag form !!!!
imgpkg push -b {harbor-url}/{project-name}/fluxcd-helm-controller-bundle:0.17.2 -f fluxcd-helm-controller/controller-yaml

docker load -i 
docker tag 641a4d6fd686 {harbor-url}/{project-name}/fluxcd-helm-controller:0.17.2
docker push {harbor-url}/{project-name}/fluxcd-helm-controller:0.17.2

kubectl create ns tap-extra-packages

vi fluxcd-helm-controller/metadata.yaml # modidy {mega-harbor-url}/{project-name}/fluxcd-helm-controller-bundle:0.17.2 to the real one..

# kubectl apply -f https://raw.githubusercontent.com/vmware-tanzu/community-edition/main/addons/packages/fluxcd-helm-controller/metadata.yaml -n tanzu-package-repo-global
kubectl apply -f fluxcd-helm-controller/metadata.yaml

# kubectl apply -f https://raw.githubusercontent.com/vmware-tanzu/community-edition/main/addons/packages/fluxcd-helm-controller/0.17.2/package.yaml -n tap-extra-packages
kubectl apply -f fluxcd-helm-controller/package.yaml

tanzu package install -n tap-extra-packages fluxcd-helm-controller -p helm-controller.fluxcd.community.tanzu.vmware.com -v 0.17.2
```

### Configuring a Helm Repository

Once we have FluxCD installed, we can create a HelmRepository resource to allow discovery and usage of the relevant charts. For this example we will use a demo repo i have hosted on github:

```bash
kubectl apply -f example/helm-repo.yaml
```

### Create the custom Cartographer config template

The next step is to create the cartographer resources to allow us to utilize the helm controller, in our platform.

```bash
kubectl apply -f config-template-helm.yaml
```

### Creating the custom supply chain
The first step is to fill in the values.yaml file in this repo based on your environment. These can be the same values as you have supplied for the ootb_supply_chain section in your tap installation values.

Once you have set the values in the file, we can now render and apply the supply chain manifest:
```bash
ytt -f values.yaml -f supply-chain.yaml | kubectl apply -f -
```

### Adding the needed RBAC
As we are adding a new resource type, that will be stamped out by the platform, we will need to add the relevant RBAC rules in our runtime and iterate clusters. The easiest way is to add a ClusterRole with the label 'apps.tanzu.vmware.com/aggregate-to-deliverable: "true"'. By using this label, we tell kubernetes to aggregate this role into the OOTB deliverable ClusterRole which we can use to give developers access to in the platform.
```bash
kubectl apply -f cluster-role.yaml
```

### Testing out the new supply chain
To test out the new supply chain, we can use the example workload in this repo
```bash
tanzu apps workload apply -f example/workload.yaml
```

### Configurable Parameters on a workload
| Parameter      | Required | Field type | Description                                                                                                                                                                                                    | Example                                                                                                      |
|----------------|----------|------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| chart_name     | yes      | string     | Name of the helm chart to use                                                                                                                                                                                  | java-web-app                                                                                                 |
| chart_version  | no       | string     | The Chart version to use.<br>If not defined, the latest version will be used                                                                                                                                   | 0.1.1                                                                                                        |
| chart_repo     | yes      | object     | The reference to a FluxCD HelmRepository CRD where the chart exists.<br>Defining the namespace is optional.<br>When not defined it will default to the workloads namespace                                     | kind: HelmRepository<br>name: demo-repo<br>namespace: helm-system                                            |
| image_key_path | no       | string     | The helm value path where the image reference should be updated.<br>If not defined, the field will be defaulted to "image.repository".<br>If the path contains a "." in one of the keys, escape it via a "\\". | java-app.image.repository                                                                                    |
| chart_values   | no       | object     | The additional chart values you want to set for the deployment.<br>The image reference will be merged with these values at config generation time.                                                             | autoscaling:<br>&nbsp;&nbsp;enabled: true<br>&nbsp;&nbsp;minReplicas: 3<br>&nbsp;&nbsp;maxReplicas: 10<br>service:<br>&nbsp;&nbsp;type: LoadBalancer |
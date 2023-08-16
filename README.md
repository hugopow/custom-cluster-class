# custom-cluster-class
Custom Cluster Class for TKG Kubernetes Clusters to configure preKubeadmCommands

Official VMware documentation: https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.3/using-tkg/workload-clusters-cclass.html#custom-ytt

# Instructions for use below
```
cd ~/.config/tanzu/tkg/clusterclassconfigs
```
```
mkdir custom
cd custom/
mkdir overlays
cd overlays/
```
```yaml
cat custom/overlays/kernels.yaml
#@ load("@ytt:overlay", "overlay")

#@overlay/match by=overlay.subset({"kind":"ClusterClass"})
---
apiVersion: cluster.x-k8s.io/v1beta1
kind: ClusterClass
metadata:
  name: tkg-vsphere-default-v1.1.0-extended
spec:
  variables:
  - name: sysctl
    required: false
    schema:
      openAPIV3Schema:
        type: string
  patches:
  - name: sysctl
    definitions:
      - selector:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          matchResources:
            machineDeploymentClass:
              names:
              - tkg-worker
        jsonPatches:
          - op: add
            path: /spec/template/spec/preKubeadmCommands/-
            value: sysctl -w vm.nr_hugepages=0
          - op: add
            path: /spec/template/spec/preKubeadmCommands/-
            value: sysctl -w vm.max_map_count=262144
```
```yaml
cat custom/overlays/filter.yaml

#@ load("@ytt:overlay", "overlay")

#@overlay/match by=overlay.not_op(overlay.subset({"kind": "ClusterClass"})),expects="0+"
---
#@overlay/remove
```
```
cd ~/.config/tanzu/tkg/clusterclassconfigs
```
## Use the default ClusterClass manifest to generate the base ClusterClass.
```
ytt -f tkg-vsphere-default-v1.1.0.yaml -f custom/overlays/filter.yaml > default_cc.yaml
```
# Generate the custom ClusterClass.
```
ytt -f tkg-vsphere-default-v1.1.0.yaml -f custom/ > custom_cc.yaml
```

# Check the difference between the default ClusterClass and your custom one, to confirm that the changes have been applied.
```yaml
diff default_cc.yaml custom_cc.yaml

4c4
<   name: tkg-vsphere-default-v1.1.0
---
>   name: tkg-vsphere-default-v1.1.0-extended
778a779,783
>   - name: sysctl
>     required: false
>     schema:
>       openAPIV3Schema:
>         type: string
2936a2942,2957
>   - name: sysctl
>     definitions:
>     - selector:
>         apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
>         kind: KubeadmConfigTemplate
>         matchResources:
>           machineDeploymentClass:
>             names:
>             - tkg-worker
>       jsonPatches:
>       - op: add
>         path: /spec/template/spec/preKubeadmCommands/-
>         value: sysctl -w vm.nr_hugepages=0
>       - op: add
>         path: /spec/template/spec/preKubeadmCommands/-
>         value: sysctl -w vm.max_map_count=262144
```
# Install the Custom ClusterClass in the Management Cluster
# Change to TKG Management Cluster Context
```
k get clusterclasses.cluster.x-k8s.io
```
# Apply the ClusterClass manifest.
```
k apply -f custom_cc.yaml
k get clusterclass
```

# Create a new cluster
```
tanzu cluster create tkg-custom -f tkg-cluster-esx4.yaml --dry-run > tkg-custom-spec.yaml
```
# Edit cluster spec and use the custom cluster class
```
vi tkg-custom-spec.yaml
```
# Change the line at the bottom under spec.topology.class to reference the new cluster Class

Old file:
```yaml
---
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  annotations:
    osInfo: photon,3,
    tkg/plan: dev
  labels:
    tkg.tanzu.vmware.com/cluster-name: tkg-custom
  name: tkg-custom
  namespace: default
spec:
  clusterNetwork:
    pods:
      cidrBlocks:
      - 100.96.0.0/11
    services:
      cidrBlocks:
      - 100.64.0.0/13
  topology:
    class: tkg-vsphere-default-v1.1.0
```
New file:
```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: tkg-custom
  namespace: default
stringData:
  password: Vmware1!
  username: administrator@vsphere.local
---
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  annotations:
    osInfo: photon,3,
    tkg/plan: dev
  labels:
    tkg.tanzu.vmware.com/cluster-name: tkg-custom
  name: tkg-custom
  namespace: default
spec:
  clusterNetwork:
    pods:
      cidrBlocks:
      - 100.96.0.0/11
    services:
      cidrBlocks:
      - 100.64.0.0/13
  topology:
    class: tkg-vsphere-default-v1.1.0-extended
```

# Create new cluster
```
tanzu cluster create -f tkg-custom-spec.yaml
```
# Add new cluster context
```
tanzu cluster kubeconfig get tkg-custom --admin
```
# Check new changes are applied
```
k get kubeadmconfigs tkg-custom-md-0-bootstrap-n5kj6-8jkq4 -o yaml
```

# Log into the node and double check
```
ssh capv@<node-ip>

sysctl -a
```

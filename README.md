# Edge Keynote Demo for SUSECON 2022

In this repo is all of the content and instructions to duplicate the full demo hosted shown on stage for the SUSECON 2022 Keynote demo. 

Everything that you need to reproduce the full stack is included in this repo.

Contents:
- Instructions for Setup and configuration of demo (Below)
- Source code and container build for 
  - ML model 
  - Demo UI 
- Preconfigured Fleet bundles for
  - Akri
  - NeuVector
  - Demo UI
  - Redis


# Requirements

To perform this demo, you need:
- Kubernetes Cluster with [Rancher Manager installed](https://rancher.com/docs/rancher/v2.5/en/quick-start-guide/deployment/quickstart-manual-setup/)
- Computer (or VM) with USB port available
- USB Camera
- A computer with helm, kubectl, and the docker cli

Note: We did run into some problems with passing a USB camera through when using a Virtual Machine.


# Demo Preparation

This demo showcases how little work it is to onboard new clusters and their peripherals when managing remote locations. Due to this, there is some initial setup required for both the new Elemental operator as well as Fleet. Once this setup is done, replicating across multiple nodes or clusters is trivial. 


## Elemental Setup

To start off, we need to install the elemental operator and build a bootstrap iso file. A longer write up of this can be found [here](https://github.com/rancher-sandbox/rancher-node-image/blob/main/getting-started.md).

Note: Through this guide, I will assume that your current kubeconfig and context are pointing to the cluster running Rancher Manager.

### Operator install and config

You can install the operator using helm with:

```
helm -n cattle-rancheros-operator-system install \
  --create-namespace rancheros-operator \
  https://github.com/rancher-sandbox/rancheros-operator/releases/download/v0.1.0/rancheros-operator-0.1.0.tgz
```

Once the operator comes up, you need to add a MachineRegistration object with:

```
cat <<EOF | kubectl apply -f -
apiVersion: rancheros.cattle.io/v1
kind: MachineRegistration
metadata:
  name: default
  namespace: default
spec:
  cloudConfig:
    rancheros:
      install:
        device: /dev/nvme0n1
    users:
    - name: root
      passwd: root
EOF
```

This will setup Rancher to allow registrations of new servers. The cloudConfig section allows you to add cloud-init configuration if needed.


Shortly after applying, if you look at the object with `kubectl get MachineRegistrations -A -o yaml` , it should have some status fields attached that look like:

```
status:
  ...
  registrationToken: <token>
  registrationURL: https://<donthackme>/v1-rancheros/registration/<token>
```


This registration url allows us to build a bootstrap iso that knows how to call home to it.

### Bootstrap ISO

Next, curl the registration url and build script(s) with:

```
REGISTRATION_URL=`kubectl get machineregistration default -ojsonpath="{.status.registrationURL}"`
curl -s -o reg.yaml $REGISTRATION_URL

curl -sLO https://raw.githubusercontent.com/rancher-sandbox/rancher-node-image/main/Dockerfile
curl -sLO https://raw.githubusercontent.com/rancher-sandbox/rancher-node-image/main/elemental-iso-build
```

Then run the build with:
```
docker build ./Dockerfile -t local/elemental-node-image
bash elemental-iso-build local/elemental-node-image iso ./reg.yaml
```

This will drop an iso file into the working directory. Burn this to a usb-stick.

### Fleet set up

Go to the "Continuous Delivery" tab in Rancher Manager. 

At the top right, select the `default` namespace (not `fleet-default`).

Click on "Cluster Groups" in the left column.

Click "Create" in the top right and name it "edge-demo".

Under "Cluster Selectors", click "Add Rule" and fill in the new line with
- Key: `edge.demo`
- Operator: `in list`
- Value: `true`

Click `Create` in the bottom right.


Then click on `Git Repos` in the left column.

Click `Add Repository` and name it `edge-demo` as well. 

Fill in the form with the following values:
- Repository URL: `https://github.com/agracey/edge-demo.git`
- Branch Name: `main`
- Paths (click `Add Path`):
  - `akri-chart`
  - `ui-yaml`
  - `redis`
  - `neuvector`
- Deploy To: `edge-demo`

Click `Create`.


## Bootstrap Server

Now that the config is done and we have a bootstrap iso image on a USB drive, let's bootstrap the node.

*NOTE: This will format the device being booted!*

Insert drive and boot server. You may need to either change your boot device order or press F10 while booting to boot into your drive.

Press enter on the Grub menu to start the installation. Once the installation is done, your device will power off. 

Note: At this time, you could ship this server anywhere and when it turns back on you would be able to provision it with the kubernetes of your choice.

Reboot the server.

## Provision Cluster 

Note: There will be a UI built to make this much simpler later this year. 

### Create Cluster

We can create a blank cluster with the following command:

```
cat <<EOF | kubectl apply -f -
apiVersion: provisioning.cattle.io/v1
kind: Cluster
metadata:
  name: edge-demo
  namespace: default
  labels:
    edge.demo: 'true'
spec:
  rkeConfig: {}
  kubernetesVersion: v1.21.9+k3s1
EOF
```


### Add Machine to Cluster

In Rancher, browse to the local cluster. 

In the left column, click `More Resources` then `rancheros.cattle.io` and finally `MachineInventories`.

You should see the server that you bootstrapped in the list. Click the drop down menu on the right side of the row and the `Edit YAML`

In this YAML, edit the following fields:

```
clusterName: edge-demo
config
  role: server
```

Click `Save`

This will trigger the bootstrapped node to be provisioned as the control plane of a new cluster. This might take a couple minutes depending on network speed. You can follow the progress from the Cluster Management section of Rancher.

## Plug in Camera

Once your cluster comes up, Fleet will automatically schedule the workload to it. This workload includes Akri which knows how to discover devices. With the configuration specified in this git repo, it will detect USB devices that can hook into the v4l2 engine and have the `:discover:` capability set. A basic webcam should work just fine!

To see the demo in action, click on the `edge-demo` cluster then Workload and Pods. Under the `Akri` namespace, you should see a few pods already. 

Go ahead and plug in your camera. In a few moments, you should see a new pod get added that knows how to use the camera you just plugged in! After the container image is scheduled and pulled down (which is large and might take some time), you should see your camera activate. 


Browse to the ip address of your cluster (can be found with `kubectl get service`) at port 8080 to see the demo app. If you show your camera a chameleon plushie it will show the picture it took along with what it matched in the page.

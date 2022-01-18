# Red Hat OpenShift demo
This repository contains the required tasks to install and configure Red Hat OpenShift Service Mesh in a OCP cluster, and deploy an example application.

## Prerequisites
 - OCP cluster up and running with version 4.6 or higher.
 - OC cli installed.

## Installation and basic configuration
### Installing the operators:

Jaeger
```bash
oc apply -f 1-ossm-operators/jaeger-operator.yaml
```

Kiali
```bash
oc apply -f 1-ossm-operators/kiali-operator.yaml
```

OSSM
```bash
oc apply -f 1-ossm-operators/ossm-operator.yaml
```

```bash
oc get clusterserviceversions.operators.coreos.com
---
NAME                         DISPLAY                                          VERSION   REPLACES                     PHASE
jaeger-operator.v1.28.0      Red Hat OpenShift distributed tracing platform   1.28.0                                 Succeeded
kiali-operator.v1.36.7       Kiali Operator                                   1.36.7    kiali-operator.v1.36.6       Succeeded
servicemeshoperator.v2.1.1   Red Hat OpenShift Service Mesh                   2.1.1-0   servicemeshoperator.v2.1.0   Succeeded
```

### Installing the Service Mesh Control Plane
Now, the operators are installed and it is time to install the Service Mesh Control Plane with the configuration desired. For this, set the [SMCP Configuration](./2-smcp/basic.yaml) file up with your preferences and install it.

Create the istio-system namespace
```
oc new-project istio-system
```

Install the Service Mesh Control Plane
```
oc apply -f 2-ossm/basic.yaml
```

You can check the installation running
```
oc get smcp -n istio-system
----
NAME    READY   STATUS            PROFILES      VERSION   AGE
basic   10/10   ComponentsReady   ["default"]   2.1.1     2m10s
```

## Adding services to the mesh
There are two ways for joining namespaces to the Service Mesh: SMMR and SMM.
### Openshift Service Mesh member roll (SMMR)

The *ServiceMeshMemberRoll* object list the projects that belong to the Control Plane. Any project that is not set in this object, is threated as external to the Service Mesh. This object must exist in the Service Mesh with name *default*.

Create the SMMR
```
oc apply -f 2-ossm/smmr.yaml
```

A namespace called *my-awesome-project* exists in the OCP cluster and it will be joined to the Service Mesh:
```bash
oc get smmr default -oyaml
---
NAME      READY   STATUS       AGE
default   1/1     Configured   5s
```

### Openshift Service Mesh member
Using this object, users who don't have privileges to add members to the *ServiceMeshMemberRoll* (e.g. users who can't access the Control Plane's namespace) can join their namespaces to the Service Mesh. But, these users need the *mesh-user* role.

If you try to create the Service Mesh Member object, you will receive the following error:
```
oc apply -f 2-smcp/smm-2.yaml 
----
Error from server: error when creating "2-smcp/smm-2.yaml": admission webhook "smm.validation.maistra.io" denied the request: user '$user' does not have permission to use ServiceMeshControlPlane istio-system/basic
```

Grant user permissions to access the mesh by granting the *mesh-user* role:
```
oc policy add-role-to-user -n istio-system --role-namespace istio-system mesh-user $user
```

This use case will be use in the application deployment step.

## Deploying the bookinfo example application
Service Mesh configuration for bookinfo sample application with external ratings database using an egress Gateway for routing TCP traffic. The bookinfo application will be deployed in two namespaces simulating front and back tiers.

Three MySQL instances are deployed outside the Mesh in the _ddbb_ project: mysql-1, mysql-2 and mysql-3. Each mysql instance has a different rating number that will be consumed by the ratings application:
* mysql-1: Ratings point equals 1.
* mysql-2: Ratings point equals 5.
* mysql-3: Ratings point equals 3.

Thus, the traffic will be balanced between the different MySQL instances.

Please, follow the steps indicated in this [repository](https://github.com/fperearodriguez/rhossm-egress-examples/tree/master/bookinfo-mysql-multiple-ns).
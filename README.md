# awss3operator - OLM Flows

## General Prereqs
- OCP 4.0 cluster (this will have the marketplace operators already installed).
- python36 and pip3 to install and build the *operator-courier*
- Docker
- An operator repo in quay.io (you can build your own) but for most examples we are going to be using 
  [aws-s3-provisioner repo](https://quay.io/repository/screeley44/aws-s3-provisioner)
  and
  image: quay.io/screeley44/aws-s3-provisioner:v1.0.0
- Create a quay.io account.

## Setup for OperatorHub Catalog
This documents the nuances for building a simple operator and then applying the operator to your local instance of OCP 4.X.
This repo is a simple example of how to do this.



#### Create your Application Repo in quay.io.
1. From the quay.io UI, click on the new repository link and select *application repository* (not image)
2. Name it your username for *Namespace* (i.e. screeley44) and then your repo name (i.e. myoperator)

#### Create your Application Bundle.
The application bundle contains the CusterResourceDefinitions (CRDs), ClusterServiceVersions (CSVs) and the Package yamls. The structure of the manifests directory allows for some freedom, but the main rule is that you must place all owned CRDs in the same directory as your CSV. The structure from this repo is a good example to follow:
```
        --> Repo Root dir
           --> manifests dir
              --> version dir 1.0
                 --> CRD1.yaml
                 --> CRD2.yaml
                 --> CSV1.0
              --> version dir 2.0
                 --> CRD1.yaml
                 --> CRD2.yaml
                 --> CSV2.0                 
```

The quay.io appregistry will be used and to push your bundle to the registry and this example will
use the [operator-courier application](https://github.com/operator-framework/operator-courier/#usage) to do. If you don't
want to build your own example, just simply clone this repo whereever you are building your example and modify a few files.

1. After building your manifests directory structure and files, the most imporant thing to note is that for now,
   you need the packageName: field from the *operator-package.yaml* to match the name of your quay.io application repo.

```yaml
packageName: awss3-operator-registry <-- This needs to match your app repo name!
channels:
- name: alpha
  currentCSV: awss3operator.1.0.0 <-- <operatorname>.<release version>
```

2. Also make sure all CRD's are in the same directory as the CSV.

#### Verify and Push the Bundle to Quay.io appregistry.

1. Run the operator-courier to verify your bundle

      operator-courier --verbose verify <manifests dir>

```
# cd <repo root>/awss3operator
# operator-courier --verbose verify ./manifests/
```

2. Generate the Authorization Token to your app registry. To do this you need to send a request to the
quay.io login api server with your quay account information.

```
# AUTH_TOKEN=$(curl -sH "Content-Type: application/json" -XPOST https://quay.io/cnr/api/v1/users/login -d '
{
    "user": {
        "username": "youraccount",
        "password": "yourpasswd"
    }
}' | jq -r '.token')


# echo $AUTH_TOKEN
basic c2NyZWVsZXk0NDpDYXQ0NHRpbSE=
```

3. If that is clean, then we can push this to the appregistry of quay.io.

      operator-courier push takes 5 parameters
      - manifests dir 
      - quay account 
      - quay app registry 
      - version release 
      - Auth Token

```
# operator-courier --verbose push ./manifests/ screeley44 awss3-operator-registry 1.0.0 "$AUTH_TOKEN"
```
*[NOTE]* Make sure to use the quotes around $AUTH_TOKEN!

4. Verify your packages in quay.io

You can follow this link and look for your package based on your quay account:
       [https://quay.io/cnr/api/v1/packages](https://quay.io/cnr/api/v1/packages)


#### Link to Openshift OperatorHub on Your Cluster

1. To do this we simply need to create an OperatorSource and apply it to the cluster.

```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorSource
metadata:
  name: community-operators-test  <-- our name for our source
  namespace: openshift-marketplace
spec:
  type: appregistry
  endpoint: https://quay.io/cnr
  registryNamespace: screeley44 <-- our quay account/Namespace that we pushed our bundle to in appregistry
  displayName: "Community Operators Test"
  publisher: "Red Hat"

```

This generates a CatalogSourceConfig automatically

```
# oc get catalogsourceconfig -n openshift-marketplace
NAME                       STATUS      MESSAGE                                       AGE
certified-operators        Succeeded   The object has been successfully reconciled   7d22h
community-operators        Succeeded   The object has been successfully reconciled   7d22h
community-operators-test   Succeeded   The object has been successfully reconciled   96m   <-- this is US!
redhat-operators           Succeeded   The object has been successfully reconciled   7d22h


# oc get catalogsourceconfig community-operators-test -n openshift-marketplace -o yaml
apiVersion: operators.coreos.com/v1
kind: CatalogSourceConfig
metadata:
  creationTimestamp: "2019-05-07T18:59:36Z"
  finalizers:
  - finalizer.catalogsourceconfigs.operators.coreos.com
  generation: 3
  labels:
    opsrc-datastore: "true"
    opsrc-owner-name: community-operators-test
    opsrc-owner-namespace: openshift-marketplace
  name: community-operators-test
  namespace: openshift-marketplace
  resourceVersion: "2866012"
  selfLink: /apis/operators.coreos.com/v1/namespaces/openshift-marketplace/catalogsourceconfigs/community-operators-test
  uid: 425f1dc4-70fa-11e9-8451-0e3aa7188248
spec:
  csDisplayName: Community Operators Test
  csPublisher: Red Hat
  packages: awss3-operator-registry  <-- this is the package it picked up
  targetNamespace: openshift-marketplace
status:
  currentPhase:
    lastTransitionTime: "2019-05-07T18:59:37Z"
    lastUpdateTime: "2019-05-07T18:59:37Z"
    phase:
      message: The object has been successfully reconciled
      name: Succeeded
  packageRepositioryVersions:
    awss3-operator-registry: 1.0.0

```

2. Now from the OpenShift Console - you can navigate to the Catalog and see your operator in the OperatorHub.
   Install the operator and you should see it running (it will prompt you for the namespace you want to run it in)

```
# oc get pods
NAME                                  READY   STATUS    RESTARTS   AGE
aws-s3-provisioner-57f6688b56-2v64b   1/1     Running   0          7m24s
```

Voila!!


**The following section discusses another way to add your app to the catalog - it might need to flush out some details, but it
is close and gives another way to deploy apps in a less formaal manner!**



## Setup and Building the Registry and Operator - Developer Catalog

#### Prerequisites and some things to note

1. Create a quay.io account (if not already done)
2. Create the following repository in your quay account repo (i.e. quay.io/jdoe/)
- aws-s3-operator-registry
3. If you want to build your own aws-s3-provisioner image you can do that as well, but for now you can use screeley44 for the actual provisioner image.
4. packageName field from <operator>.package.yaml (awss3operator.package.yaml) should match the repository name where your actual operator/controller image is stored (i.e. aws-s3-provisioner)

#### Clone Framework Repos

1. Clone the Operator repo
```
# cd <gopath>/src/github.com/yard-turkey
# git clone https://github.com/yard-turkey/aws-s3-olm-operator.git

```

2. Create Operator-Framework Dir structure in *<gopath>/src/github.com*
```
 # cd /<gopath>/src/github.com
 # mkdir operator-framework
 # cd operator-framework 
 
```

3. Clone the Operator SDK in the Operator-Framework directory path.

```
 # git clone https://github.com/operator-framework/operator-sdk.git
```


4. Clone the Operator-Registry in the Operator-Framework directory path.

```
 # git clone https://github.com/operator-framework/operator-registry.git
``` 


#### Build the Registry base image and builder container

1. From the operator-registry repo run the following commands.

```
 # cd operator-registry
 # docker build -t quay.io/operator-framework/upstream-registry-builder:latest -f  upstream-builder.Dockerfile .
 
```

2. Now build the server-registry and push it.
```
 # cd ../../yard-turkey-aws-s3-olm-operator
 # docker build --no-cache -t quay.io/<quay account>/aws-s3-operator-registry:v1.0.0 -f upstream-Dockerfile .
 # docker push quay.io/<quay account>/aws-s3-operator-registry:v1.0.0
 
 i.e.
 # docker build -t quay.io/screeley44/aws-s3-operator-registry:v1.0.0 -f upstream-Dockerfile .
 # docker push quay.io/screeley44/aws-s3-operator-registry:v1.0.0
```

#### Install the Catalog and Operator on your OCP 4.0 cluster.

1. Create the Catalog. (Note if you are not using the screeley44 account, you will need to update the catalog-source.yaml)

```
 # oc create -f ./manifests/awss3operator/catalog-source.yaml
```

2. Make sure the catalog is running in the *openshift-operator-lifecycle-manager* project/namespace.

```
 # oc get pods -n openshift-operator-lifecycle-manager
```

3. Then create the Operator Group using operator-group.yaml
```
 # oc create -f ./manifests/awss3operator/operator-group.yaml
```

4. Next we can create a subscription - you can do this from the console or create the subscription.yaml

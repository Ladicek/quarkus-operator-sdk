## Spamming your Kubernetes cluster with jokes since 2021! :heart:

The idea is that you create a `JokeRequest` custom resource that you apply to your cluster. The
operator will do its best to comply and create a `Joke` custom resource on your behalf if everything
went well. Jokes are retrieved from the https://v2.jokeapi.dev/joke API endpoint. The request can be
customized to your taste by specifying which category of jokes you'd like or the amount of
explicitness / topics you can tolerate. You can also request a "safe" joke which should be
appropriate in most settings.

### Quick start

To quickly test your operator (and develop it), no need to create an image and deploy it to the
cluster. You can just follow the steps below to get started quickly:

- Connect to your cluster of choice using `kubectl/oc`, select the appropriate namespace/project.
  The operator will automatically connect to that cluster/namespace combination when started.
- Run `mvn -Dquickly -Dquarkus.operator-sdk.crd.apply=true` on the parent directory to build the
  project locally and move to this
  sample directory.
- Run `mvn package` to build the operator. This will automatically generate several resources for
  you, in particular the CRDs associated with the custom resources we will be dealing with. These
  CRDs are generated in `target/kubernetes/` and come in `v1` version which correspond to the
  versions of the CRD spec. Note that older clusters might need the version `v1beta1` which is no
  longer generated by default. To generate it, you need to use the
  property `quarkus.operator-sdk.crd.versions=v1beta1,v1`, so both versions `v1` and `v1beta1` are
  generated.
  - Required CRDs should be applied automatically to your cluster (provided you have the
    permission to do so, which might require to log in as an administrator-level user on OpenShift)
    because the `quarkus.operator-sdk.crd.apply` property has been set to `true`. Additionally,
    the `quarkus.operator-sdk.crd.generate-all` property has been set to `true` as
    well in `application.properties`, which means that CRDs will also be generated for 3rd party
    Custom Resource (which the `Joke` CR is in this example).
- Launch the app in dev mode: `mvn quarkus:dev`
- Deploy the test request (or your own): `kubectl apply -f src/main/k8s/jokerequest.yml`. The operator will take your request and attempt to retrieve a joke from the api. If everything went well, a `Joke` resource named after the `id` of the joke retrieved from the API will be created on your cluster.
- You can check the status of the request by doing something similar to, `jr` being the short name associated with `JokeRequest`:
    ```sh
    kubectl describe jr
    ```
- You can check if your joke has been created and look for the id in the returned list:
    ```sh
    kubectl get jokes
    ```
  - Check your joke:
    ```sh
    kubectl get jokes <your joke id> -o jsonpath="{.joke}{'\n'}" 
    ```

### Deployment

This section explains how to deploy your operator using the [Operator Lifecycle Manager (OLM)](https://olm.operatorframework.io/) by following the next steps:

0. Requirements

Make sure you have installed the [opm](https://github.com/operator-framework/operator-registry) command tool and have connected to a Kubernetes cluster with the OLM installed.

1. Generate the Operator image and bundle manifests

This example uses the [Quarkus Jib container image](https://quarkus.io/guides/container-image#jib) extension to build the Operator image. 
Also, the Quarkus Operator SDK provides the `quarkus-operator-sdk-bundle-generator` extension that generates the Operator bundle manifests at `target/bundle/<operator name>`.
So, you simply need to run the next Maven command to build and push the operator image, and also to generate the bundle manifests:

```shell
mvn clean package -Dquarkus.container-image.build=true \
    -Dquarkus.container-image.push=true \
    -Dquarkus.container-image.registry=<your container registry. Example: quay.io> \
    -Dquarkus.container-image.group=<your container registry namespace> \
    -Dquarkus.kubernetes.namespace=<the kubernetes namespace where you will deploy the operator> \
    -Dquarkus.operator-sdk.bundle.package-name=<the name of the package that bundle image belongs to> \ 
    -Dquarkus.operator-sdk.bundle.channels=<the list of channels that bundle image belongs to>
```

For example, if we want to name the package `joke-operator` and use the `alpha` channels, we would need to append the properties `-Dquarkus.operator-sdk.bundle.package-name=joke-operator -Dquarkus.operator-sdk.bundle.channels=alpha`.

---
**NOTE**
Find more information about channels and packages in [this link](https://olm.operatorframework.io/docs/best-practices/channel-naming/#channels).
---

---
**NOTE**
If you're using an insecure container registry, you'd also need to append the next property to the Maven command `-Dquarkus.container-image.insecure=true`.
---

2. Build the Operator Bundle image

An Operator Bundle is a container image that stores Kubernetes manifests and metadata associated with an operator. You can find more information about this in [here](https://olm.operatorframework.io/docs/tasks/creating-operator-bundle/). 
In the previous step, we generated the bundle manifests at `target/bundle/<operator name from the CSV Metadata>` which includes a ready-to-use Dockerfile with name `bundle.Dockerfile` file that you will use to build and push the final Operator Bundle image to your container registry:

```shell
JOKE_BUNDLE_IMAGE=<your container registry>/<your container registry namespace>/<bundle image name>:<tag>
docker build -t $JOKE_BUNDLE_IMAGE -f target/bundle/<operator name>/bundle.Dockerfile target/bundle/<operator name>
docker push $JOKE_BUNDLE_IMAGE
```

For example, if we want to name our bundle image as `joke-manifest-bundle`, our container registry is `quay.io`, our Quay user is `myuser` and the tag we're releasing is `1.0`, the final `JOKE_BUNDLE_IMAGE` property would be `quay.io/myuser/joke-manifest-bundle:1.0`. 

3. Make your operator available within a Catalog

OLM uses catalogs to discover and install Operators and their dependencies. So, a catalog is similar to a repository of operators and their associated versions that can be installed on a cluster. Moreover, the catalog is also a container image that contains a collection of bundles and channels. Therefore, we'd need to create a new catalog (or update an existing one if you're already have one), build/push the catalog image and then install it on our cluster.

So far, we have already built the Operator bundle image at `$JOKE_BUNDLE_IMAGE` (see above) and next, we need to add this Operator bundle image into our catalog. For doing this, we'll use the `olm` tool as follows:

```shell
CATALOG_IMAGE=<catalog container registry>/<catalog container registry namespace>/<catalog name>:<tag>
opm index add \
    --bundles $JOKE_BUNDLE_IMAGE \
    --tag $CATALOG_IMAGE \
    --build-tool docker
docker push $CATALOG_IMAGE
```

For example, if our catalog name is `joke-catalog`, our container registry for the catalog is `quay.io`, our Quay user is `myuser` and the container tag we're releasing is `59.0`, the final `CATALOG_IMAGE` property would be `quay.io/myuser/joke-catalog:59.0`.

---
**NOTE**
If you're using an insecure registry, you'd need to append the argument `--skip-tls` to the `opm index` command.
---

Once we have our catalog image built and pushed at `$CATALOG_IMAGE` image, we need to install it on our cluster using the `CatalogSource` resource by doing the next command:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: joke-catalog-source
  namespace: operators
spec:
  sourceType: grpc
  image: $CATALOG_IMAGE
EOF
```

---
**NOTE**
We need to install the catalog source at the Kubernetes namespace where we installed OLM, which by default it's `operators`.
---

Once the catalog is installed, you should see the catalog pod up and running:

```shell
kubectl get pods -n <K8S NAMESPACE> --selector=olm.catalogSource=joke-catalog-source
```

4. Install your operator via OLM

OLM deploys operators via [subscriptions](https://olm.operatorframework.io/docs/tasks/install-operator-with-olm/#install-your-operator). Creating a  `Subscription` will trigger the operator deployment. You can simply create the `Subscription` resource that contains the operator name and channel to install by running the following command:

```shell
cat <<EOF | kubectl create -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: joke-subscription
  namespace: <Kubernetes namespace where the Joke operator will be installed>
spec:
  channel: alpha
  name: joke
  source: joke-catalog-source
  sourceNamespace: operators
EOF
```

---
**NOTE**
We'll install the operator in the target namespace defined at the metadata object. The `sourceNamespace` value is the Kubernetes namespace where the catalog was installed on (`operators` by default, see the previous step).
---

Once the subscription is created, you should see your operator pod up and running:

```shell
kubectl get csv -n operators jokerequestreconciler
```

Also, this example needs the `Joke` CRD to be installed on the cluster:

```shell
kubectl apply -f src/main/k8s/jokes.samples.javaoperatorsdk.io-v1.yml
```

5. Test your operator

Deploy the test request (or your own): `kubectl apply -n <your namespace> -f src/main/k8s/jokerequest.yml` and,
if everything went well, a `Joke` resource named after the `id` of the joke retrieved from the API will be
  created on your cluster: `kubectl get jokes -n <your namespace>`.

### Native binary

To build a native binary for your platform, just run: `mvn package -Pnative`. The binary will be
found in the `target` directory.

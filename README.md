Istio Service to Service Authentication and Authorization demonstration
=======================================================================

Local Environment preparation
-----------------------------

1. Set the following environment variables to help the demonstration process:

```
export BILLING_ACCOUNT=<billing_account>
export PROJECT_ID=<project_id>
export ZONE=europe-west2-a
```

Infrastructure Deployment steps
-------------------------------

### Create a project

1. Create a project and link it with a billing account:

```
gcloud projects create $PROJECT_ID --no-enable-cloud-apis
gcloud beta billing projects link $PROJECT_ID --billing-account=$BILLING_ACCOUNT
```

### Provision a GKE cluster

1. Enable the GKE API and create a cluster with `gcloud` on a project with a default VPC:

```
gcloud --project $PROJECT_ID services enable container.googleapis.com
gcloud --project $PROJECT_ID container clusters create k0 --zone $ZONE --machine-type e2-standard-2 --release-channel=rapid
```

Note: this process is known to work with GKE 1.17 and 1.18. This may change over time.

2. Get the cluster's credentials:

```
gcloud --project $PROJECT_ID container clusters get-credentials k0 --zone $ZONE
```

Software deployment steps
-------------------------

### Deploying Open-Source Istio

On your laptop, on bash/zsh terminal, follow the [Getting Started](https://istio.io/latest/docs/setup/getting-started/)
guide and install Istio, the sample Bookinfo application and Kiali Dashboard or follow the following steps:

1. Download Istio:

```
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.6.8 TARGET_ARCH=x86_64 sh -
cd istio-1.6.8
```

2. Install Istio on the GKE cluster with the demonstration profile:

```
export PATH=$PWD/bin:$PATH
istioctl install --set profile=demo
```

3. Enable Istio's Envoy proxy sidecar container injection into Pods:

```
kubectl label namespace default istio-injection=enabled
```

4. Install the Bookinfo sample application and wait for it to come online (use CTRL+c to exist the `watch` command):

```
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl get service
watch -n 5 kubectl get pods
```

5. Make sure Istio installed correctly and fetch a few details of its configuration to help on follow-up steps:

```
istioctl analyze
kubectl get svc istio-ingressgateway -n istio-system
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```

6. Install bundled Istio addons including the Kiali dashboard. A few of the steps are meant to work around errors
deploying Grafana:

```
kubectl apply -f samples/addons
kubectl apply -f samples/addons
kubectl delete -f samples/addons/grafana.yaml
kubectl apply -f samples/addons/grafana.yaml
kubectl apply -f samples/addons
```

7. Wait for Kiali to be deployed and open a connection to it locally:

```
while ! kubectl wait --for=condition=available --timeout=600s deployment/kiali -n istio-system; do sleep 1; done
istioctl dashboard kiali &
cd ..
```

Service to Service Authentication and Authorization configuration on Istio
--------------------------------------------------------------------------

1. Navigate through Kiali dashboard to get a sense of what it provides.

2. Verify that Bookinfo is up and running the following command and using the result on your browser:

```
echo http://"$GATEWAY_URL/productpage"
```

3. Check Kiali dashboard's graph to see the new additions to it.

4. Generate some traffic throughout the demonstration:

```
for i in $(seq 1 10000); do curl -s http://${GATEWAY_URL}/productpage -o /dev/null; sleep 0.5; done &
```

5. Show Kiali details such as RPS, latency, etc., through the Graph section.

### Set Peer Authentication to MTLS Permissive

We start with lax security to restrict it as the demonstration goes.

1. Explicitly set the mesh's authentication policy to `PERMISSIVE` (this is the default behaviour on Istio 1.6.8 and
therefore the cluster behaviour is not changed). This allows for authenticated and un-authenticated communication to
happen:

```
cat PeerAuthentication-Permissive.yaml
kubectl apply -f PeerAuthentication-Permissive.yaml
```

2. Notice on Kiali's graph that the mTLS "lock" icon is shown when the "Security Badges" are turned on.

### Set Peer Authentication to MTLS Strict

1. Set the mesh's authentication policy to `STRICT`. This configuration will block any un-authenticated communication
between services from happening:

```
cat PeerAuthentication-Strict.yaml
kubectl apply -f PeerAuthentication-Strict.yaml
```

On Bookinfo running on Istio 1.6.8 service mesh, this does not change any behaviour. It just makes sure other workloads
running on the service mesh are not communicating in an un-authenticated way.

### Enable Authorization between microservices

Authorization enforcement is disabled by default on the Istio service mesh 1.6.8. 

1. To enable Authorization enforcement on the `default` namespace a `AuthorizationPolicy` object needs to be create as
shown below:

```
cat deny-all-AuthorizationPolicy.yaml
kubectl apply -f deny-all-AuthorizationPolicy.yaml
```

2. Confirm that after setting the default namespaces's `AuthorizationPolicy` to deny all that the application is no
longer available:

With the command-line:
```
curl -v http://${GATEWAY_URL}/productpage
```

Or on your browser:
```
echo http://${GATEWAY_URL}/productpage
```

The `RBAC: access denied` message should be returned. curl should also return a `403 Forbidden` code.

On the Kiali dashboard you will see communication originating from the `istio-ingressgateway` being blocked at the
`productpage` microservice. The Kiali dashboard graph will show the arrow connecting the gateway and the app turn yellow
and red as the success call rate drops from 100% to 0%. You will also notice that the arrow depicting calls made by the
`productpage` microservice to other microservices, and arrows from these microservices to others downstream become grey.
This is to be expected because as `productpage` is rejecting all calls, no new calls are being made to microservices
downstream.

3. To authorize communication incoming from `istio-ingressgateway` to the `productpage` microservice, we need to create
an `AuthorizationPolicy` object with a label selector that is more specific than the `deny-all` `AuthorizationPolicy`
created on the previous step. This can be done as follows:

```
cat allow-all-productpage-viewer-AuthorizationPolicy.yaml
kubectl apply -f allow-all-productpage-viewer-AuthorizationPolicy.yaml
```

The application should once again be available:

Test with the command-line:
```
curl -v http://${GATEWAY_URL}/productpage
```

Test with your browser:
```
echo http://${GATEWAY_URL}/productpage
```

The `productpage` should be shown but notice that the details and the product reviews are not being shown. That's
because the `productpage` cannot request anything from those microservices.

On the Kiali dashboard you will see:
- the arrow between `istio-ingressgateway` and `productpage` turn from red to yellow then to green as the calls cease to
be dropped.
- the arrows between `productpage` and both `details` and `reviews` microservices turn from grey to red as the calls are
dropped due to lack of authorization.

4. Create the `AuthorizationPolicy` objects which allows the `productpage` microservice to access the `details`
microservice:

```
cat allow-productpage-details-viewer-AuthorizationPolicy.yaml
kubectl apply -f allow-productpage-details-viewer-AuthorizationPolicy.yaml
```

Check the Bookinfo page with your browser:
```
echo http://${GATEWAY_URL}/productpage
```

The `details` section should no longer show an error. 

On the Kiali dashboard you will see:
- the arrow between `productpage` and `details` microservices turn from red to yellow then to green as the calls are
once again accepted.

5. Finally, create the `AuthorizationPolicy` objects to allow the remain microservices to be accessed:

```
cat allow-remaining-microservices-viewer-AuthorizationPolicy.yaml
kubectl apply -f allow-remaining-microservices-viewer-AuthorizationPolicy.yaml
```

The principals shown on the `AuthorizationPolicy` objects follow the SPIFFE ID path [1]
(`cluster.local/ns/default/sa/bookinfo-...`) are using service accounts which names start with `bookinfo-` that exist on
the `default` namespace of the local Kubernetes cluster (`cluster.local`).

[1] - [The SPIFFE Identity and Verifiable Identity Document](https://github.com/spiffe/spiffe/blob/master/standards/SPIFFE-ID.md)

Infrastructure Clean-up
-----------------------

1. Destroy the demonstration GKE cluster:

```
gcloud --project $PROJECT_ID container clusters delete k0 --zone $ZONE
```

2. Delete the test project:

```
gcloud projects delete $PROJECT_ID
```

3. Delete the temporary Istio installation files:

```
rm -rf istio-1.6.8
```

4. Unset your environment variables:

```
unset PROJECT_ID INGRESS_HOST INGRESS_PORT GATEWAY_URL ZONE
```

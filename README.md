Istio Service to Service Authentication and Authorization demonstration
=======================================================================

Environment preparation
-----------------------

1. Set the following environment variables to help the demonstration process:

```
export PROJECT_ID=<project_id>
```

Demonstration
-------------

### Provision a Cluster

1. Create a cluster with `gcloud` on a project with a default VPC:

```
gcloud --project $PROJECT_ID services enable container.googleapis.com
gcloud beta --project $PROJECT_ID container clusters create k0 --zone europe-west2-a --addons Istio --machine-type n1-standard-2
```

1. Get the cluster's credentials:

```
gcloud --project $PROJECT_ID container clusters get-credentials k0 --zone europe-west2-a
```

### Deploying Open-Source ISTIO

1. Follow the [Installing Istio on a GKE cluster](https://cloud.google.com/istio/docs/how-to/installing-oss) guide with the following mofifications.

* When installing istio's Helm template, make sure you add the `--values install/kubernetes/helm/istio/values-istio-demo.yaml` parameter as shown below:

```
helm template install/kubernetes/helm/istio \
  --name istio --namespace istio-system \
  --values install/kubernetes/helm/istio/values-istio-demo.yaml \
| kubectl apply -f -
```

1. Wait for all istio-system workloads and services to become available:

```
kubectl get all -n istio-system
```

1. When the `istio-ingressgateway` is given an `EXTERNAL-IP` set some environment variables to ease the demonstration:

```
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```

### Set Istio's starting configuration

1. Enable automatic sidecar injection

```
kubectl label namespace default istio-injection=enabled
```

### Set authentication to MTLS Permissive

We start with lax security to restrict it as the demonstration goes.

1. Set the mesh's authentication policy to `PERMISSIVE`. This allows for authenticated and unautheticated communication to happen:

```
cat MeshPolicy-Permissive.yaml
kubectl apply -f MeshPolicy-Permissive.yaml
```

### Deploy Bookinfo Demonstration Application

The bookinfo application is the main example shown on Istio's [documentation](https://istio.io/docs/examples/bookinfo/). We'll leverage it as it is well known.

#### Bookinfo Architecture Diagram

![Bookinfo Architecture Diagram][bookinfo-architecture-diagram]

#### Bookinfo Deployment Steps

1. Deploy the Bookinfo app:

```
cat bookinfo.yaml
kubectl apply -f bookinfo.yaml 
```

1. Check services and pods are running:

```
kubectl get services
kubectl get pods
```

1. Test a connection from the ratings microservice to the productpage microservice:

```
kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
```

The result should show `<title>Simple Bookstore App</title>`

1. Test a connection to the application through the Ingress Gateway:

On the command-line:
```
curl -v http://${GATEWAY_URL}/productpage
```

On the browser:
```
echo http://${GATEWAY_URL}/productpage
```

This should fail because istio's default ingress gateway is not at this point configured to receive and route traffic to bookinfo's microservices.

1. Deploy bookinfo's Gateway and VirtualService CRD objects.

```
cat bookinfo-gateway.yaml
kubectl apply -f bookinfo-gateway.yaml 
```

This selects the default istio gateway and configures it to receive HTTP requests on port 80 on any hostname. It also specifies which request paths should be considered routes to the productpage microservice.

1. Now that the ingress routing rules have been specified, test a connection to the application through the Ingress Gateway:

```
curl -s http://${GATEWAY_URL}/productpage | grep -o "<title>.*</title>"
```

The result should show `<title>Simple Bookstore App</title>`.

1. Access Bookinfo through your browser:

```
echo http://${GATEWAY_URL}/productpage
```

You should be able to see the Bookinfo webpage.

### Accessing Kiali's Dashboard

1. On a separate terminal window, proxy Kiali's dashboard to your local machine:

```
kubectl -n istio-system port-forward svc/kiali 20001:20001
```

1. Open Kiali dashboard on a browser [http://localhost:20001]()

1. Log in with `admin` as both a username and a password.

1. Navigate to the `Graph` item on the left-side menu.

1. Configure the graphic by selecting the `default` namespace,  `Versioned app graph`, `No edge labels`, all badges including `security`, `Traffic animation` and `Every 10s` refresh rate.

1. On a separate window, run the following command to generate traffic:

```
for i in $(seq 1 10000); do curl -s  http://${GATEWAY_URL}/productpage -o /dev/null; done
```

1. Watch Kiali dashboard come alive with a graph of the services and animation of the traffic flow.

### Securing the Bookinfo application

In this section we will secure the Bookinfo application by first enabling TLS authentication and enable transport-level encryption and second by enabling authorization.

#### Enable TLS Authentication and transport-level encryption

1. Change the mesh's policy to require mTLS (mutual-TLS) authentication and encryption:

```
cat MeshPolicy-Strict.yaml
kubectl apply -f MeshPolicy-Strict.yaml
```

1. Confirm that after setting the mesh's policy to STRICT that the application is no longer available:

With the command-line:
```
curl -v http://${GATEWAY_URL}/productpage
```

Or on your browser:
```
echo http://${GATEWAY_URL}/productpage
```

On both cases the `upstream connect error or disconnect/reset before headers. reset reason: connection termination` message should be returned. You should also see that curl returned a `503 Gateway Unavailable`

On the Kiali dashboard, the traffic flow arrows should become gray, indicating no traffic flowing and red if errors for productpage as it is unable to contact the microservices it depends on.

1. For the sake of demonstration lets revert the mesh-wide policy to permissive and restrict the policy at the microservices level:

```
cat MeshPolicy-Permissive.yaml
kubectl apply -f MeshPolicy-Permissive.yaml
```

1. The application should once again be available:

Test with the command-line:
```
curl -v http://${GATEWAY_URL}/productpage
```

Test with your browser:
```
echo http://${GATEWAY_URL}/productpage
```

1. Now restrict the policy at the microservice level:

```
cat policy-all-mtls.yaml
kubectl apply -f policy-all-mtls.yaml
```

1. Confirm that after setting the mesh's policy to STRICT that the application is no longer available:

With the command-line:
```
curl -v http://${GATEWAY_URL}/productpage
```

Or on your browser:
```
echo http://${GATEWAY_URL}/productpage
```

The `MeshPolicy` and `Policy` objects only control the configuration on the Envoy Proxy making the request, not the Envoy proxies receiving the request. When setting `MeshPolicy` or `Policy` objects to STRICT mTLS, this forces the Envoy proxy making the request to use mTLS authentication and transport-level encryption. Which is only possible if the receiving Envoy proxy accepts mTLS.

1. To enable the microservices' receiving Envoy proxies to accept TLS traffic, their `DestinationRules` needs to be set to allow it. Do so by:

```
cat destination-rule-all-mtls.yaml
kubectl apply -f destination-rule-all-mtls.yaml
```

The above sets the traffic policy to accept mTLS with certificates signed by Citadel (`ISTIO_MUTUAL`).

1. The application should once again be available:

Test with the command-line:
```
curl -v http://${GATEWAY_URL}/productpage
```

Test with your browser:
```
echo http://${GATEWAY_URL}/productpage
```

#### Enable Authorization between microservices

Authorization enforcement is disabled by default on the mesh. 

1. To enable Authorization enforcement on the `default` namespace a `ClusterRbacConfig` object needs to be create as shown below:

```
cat ClusterRbacConfig-On-with-Inclusions.yaml
kubectl apply -f ClusterRbacConfig-On-with-Inclusions.yaml
```

1. Confirm that after setting the default namespaces's RBAC configuration to `ON_WITH_INCLUSION` that the application is no longer available:

With the command-line:
```
curl -v http://${GATEWAY_URL}/productpage
```

Or on your browser:
```
echo http://${GATEWAY_URL}/productpage
```

The `RBAC: access denied` message should be returned. curl should also return a `403 Forbidden` code.

To enable authorization we need to first create `ServiceRole` objects for each microservice and then create `ServiceRoleBinding` for consumers of those services.

1. First create the `ServiceRole` objects as shown below:

```
cat ServiceRoles.yaml
kubectl apply -f ServiceRoles.yaml
```

1. Second create a `ServiceRoleBinding` object to allow all authenticated and un-authenticated users to access the `productpage` microservice:

```
cat ServiceRoleBindings-1.yaml
kubectl apply -f ServiceRoleBindings-1.yaml
```

1. The application should once again be available:

Test with the command-line:
```
curl -v http://${GATEWAY_URL}/productpage
```

Test with your browser:
```
echo http://${GATEWAY_URL}/productpage
```

The `productpage` should be shown but notice that the details and the product reviews are not being shown. That's because the `productpage` cannot request anything from them.

1. Enable the remaining microservices to communicate with other microservices as shown on the architecture diagram:

![Bookinfo Architecture Diagram][bookinfo-architecture-diagram]

```
cat ServiceRoleBindings-2.yaml
kubectl apply -f ServiceRoleBindings-2.yaml
```

You will notice that the user is the same (`cluster.local/ns/default/sa/default`) for all `ServiceRoleBindings`. The reason for this is that all services are using the same service account (the `default` service account for the `namespace`). Another aspect to notice is that the user ID follows the SPIFFE ID path [1].

[1] - [The SPIFFE Identity and Verifiable Identity Document](https://github.com/spiffe/spiffe/blob/master/standards/SPIFFE-ID.md)

Clean-up
--------

1. Destroy the demonstration cluster:

```
gcloud --project $PROJECT_ID container clusters delete k0 --zone europe-west2-a
```

1. Unset your configuration changes:

```
unset PROJECT_ID INGRESS_HOST INGRESS_PORT GATEWAY_URL
```
[bookinfo-architecture-diagram]: https://istio.io/docs/examples/bookinfo/withistio.svg "Bookinfo Architecture Diagram"

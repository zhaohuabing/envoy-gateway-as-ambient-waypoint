Deploy Envoy Gateway as Waypoint proxy for Istio Ambient Service Mesh.

## Prerequisites

A Kubernetes cluster, it can be a local cluster created by Kind or a remote cluster.

## Deploy Ambient Service Mesh

```bash
istioctl install --set profile=ambient -y
```

## Deploy Envoy Gateway

The

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v0.0.0-latest \
  --set config.provider.kubernetes.deploy.type=GatewayNamespace \
  -n envoy-gateway-system \
  --create-namespace
```

## Enable Ambient Service Mesh

```bash
kubectl label namespace default istio.io/dataplane-mode=ambient
```

## Deploy Test Application

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/bookinfo/platform/kube/bookinfo.yaml
```

## Deploy Ingress Gateway

```bash
kubectl apply -f manifests/ingress.yaml
```

This manifest creates a Gateway for the Ingress gateway, and an HTTPRoute to route traffic to the Bookinfo application.
Under the hood, the Ingress gateway is deployed by Envoy Gateway.

Get the name of the Envoy service created the by the example Gateway:

```bash
export ENVOY_SERVICE=$(kubectl get svc --selector=gateway.envoyproxy.io/owning-gateway-name=bookinfo-ingress -o jsonpath='{.items[0].metadata.name}')
```

Port forward to the Envoy service:

```bash
kubectl port-forward service/${ENVOY_SERVICE} 8080:80 &
```

## Test the Ingress Gateway

Open http://localhost:8080/productpage to see the Bookinfo application. In the product page, you can see the reviews for the book. If you refresh the page, you can see different versions of the reviews: with red stars, with black stars, and with no stars. This is because the Reviews service has three versions: v1, v2, and v3, and the ingress gateway is routing the traffic to different versions by default.

## Deploy Waypoint Proxy

```bash
kubectl apply -f manifests/waypoint.yaml
```

This manifest creates a Waypoint proxy for the Reviews service. It also creates a HTTPRoute to route traffic to the v1 version of the Reviews service.

Under the hood, the Waypoint proxy is deployed by Envoy Gateway.

## Test the Waypoint Proxy

Open http://localhost:8080/productpage to see the Bookinfo application. You can see that there are no stars in the reviews. This is because the Waypoint proxy is routing the traffic to the v1 version of the Reviews service.

## Apply Rate Limiting

```bash
kubectl apply -f manifests/rate-limiting.yaml
```

This manifest creates a BackendTrafficPolicy to apply rate limiting to the Reviews service. The limit is 3 request per minute.

Open http://localhost:8080/productpage to see the Bookinfo application. If you refresh the page more than 3 times, you will see that the reviews are not displayed. Instead, you will see a message "Sorry, product reviews are currently unavailable for this book."

Look at the Envoy Gateway waypoint proxy's logs, you can see that the request is rate limited:

```bash
kubectl logs envoy-default-reviews-waypoint-b1b67dfe-546f4ff4fc-b2pfj --tail 1
Defaulted container "envoy" out of: envoy, shutdown-manager
{":authority":"reviews:9080","bytes_received":0,"bytes_sent":18,"connection_termination_details":null,"downstream_local_address":"10.244.0.49:9080","downstream_remote_address":"10.244.0.44:40937","duration":0,"method":"GET","protocol":"HTTP/1.1","requested_server_name":null,"response_code":429,"response_code_details":"local_rate_limited","response_flags":"RL","route_name":"httproute/default/reviews/rule/0/match/0/reviews","start_time":"2025-05-01T08:34:27.573Z","upstream_cluster":"httproute/default/reviews/rule/0","upstream_host":null,"upstream_local_address":null,"upstream_transport_failure_reason":null,"user-agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/134.0.0.0 Safari/537.36","x-envoy-origin-path":"/reviews/0","x-envoy-upstream-service-time":null,"x-forwarded-for":"10.244.0.44","x-request-id":"e52f5442-5bb7-47bb-97e1-652f56bb6ad6"}
```

## key points:

- The gatewayClassName is set to `eg-ingress` and `eg-waypoint` to use the Envoy Gateway as the controller to deploy the ingress gateway and the waypoint proxy.
- Since the waypoint proxy deployed by Envoy Gateway does not handle HBONE, a label `istio.io/dataplane-mode: ambient` is added to the Gateway resource to indicat that the Ztunnel needs to intercept the inbound and outbound traffic and handle HBONE for the waypoint proxy, which is also known as the "sandwich mode".
- A label `istio.io/use-waypoint: eg-waypoint` is added to the Backend service to indicate that the service should use the `eg-waypoint` waypoint proxy.


## Try Envoy Gateway's Layer 7 features
All the Envoy Gateway's Layer 7 features are available, including the Gateway API features, Envoy Gateway's ClientTrafficPolicy, BackendTrafficPolicy, and SecurityPolicy.
You can follow the [Envoy Gateway's documentation](https://gateway.envoyproxy.io/docs/) to try them out.

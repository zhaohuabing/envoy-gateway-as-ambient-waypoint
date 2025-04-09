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
  --version v1.3.2 \
  --set global.images.envoyGateway.image=zhaohuabing/gateway:ambient \
  -n envoy-gateway-system \
  --create-namespace
```

## Enable Ambient Service Mesh

Label the `envoy-gateway-system` namespace to enable the ambient service mesh.

```bash
kubectl label namespace envoy-gateway-system istio.io/dataplane-mode=ambient
```

## Deploy Test Application

Create a test application to test the ambient service mesh. This manifest also creates a Gateway for the Waypoint proxy, and an HTTPRoute to route traffic to the test application. In the HTTPRoute, we add an HTTPFilter to add a header to the request to test the Envoy Gateway's Layer 7 routing.

Some key points:

- The gatewayClassName is set to `eg` to use the Envoy Gateway as the controller to deploy the waypoint proxy.
- Since the waypoint proxy deployed by Envoy Gateway does not handle HBONE, a label `istio.io/dataplane-mode: ambient` is added to the Gateway resource to indicat that the Ztunnel needs to intercept the
inbound and outbound traffic and handle HBONE for the waypoint proxy, which is also known as the "sandwich mode".
- A label `istio.io/use-waypoint: eg-waypoint` is added to the Backend service to indicate that the service should use the `eg-waypoint` waypoint proxy.

```bash
kubectl apply -f manifests.yaml
```

## Test the Envoy Gateway's Layer 7 Routing with Waypoint

```bash
kubectl -n envoy-gateway-system exec -it test-client -- curl http://backend:3000
```

You should see the response like this with the header `Add-Header: foo`:

```
{
 "path": "/",
 "host": "backend:3000",
 "method": "GET",
 "proto": "HTTP/1.1",
 "headers": {
  "Accept": [
   "*/*"
  ],
  "Add-Header": [
   "foo"
  ],
  "User-Agent": [
   "curl/7.88.1"
  ],
  "X-Envoy-External-Address": [
   "10.244.0.12"
  ],
  "X-Forwarded-For": [
   "10.244.0.12"
  ],
  "X-Forwarded-Proto": [
   "http"
  ],
  "X-Request-Id": [
   "ab1e4da7-5f1d-45ea-bb4a-fa1ccd03b82b"
  ]
 },
 "namespace": "",
 "ingress": "",
 "service": "",
 "pod": ""
}
```

You can check the Ztunnel's and Envoy proxy's logs to verify the traffic path.

```bash
kubectl -n istio-system logs -l app=ztunnel --tail 1
2025-04-09T11:44:45.641989Z	info	access	connection complete	src.addr=10.244.0.12:51504 src.workload="test-client" src.namespace="envoy-gateway-system" src.identity="spiffe://cluster.local/ns/envoy-gateway-system/sa/default" dst.addr=10.244.0.14:15008 dst.hbone_addr=10.96.113.62:3000 dst.service="backend.envoy-gateway-system.svc.cluster.local" dst.workload="envoy-envoy-gateway-system-eg-waypoint-3d55183b-ccc6f8575-gs2hp" dst.namespace="envoy-gateway-system" dst.identity="spiffe://cluster.local/ns/envoy-gateway-system/sa/envoy-envoy-gateway-system-eg-waypoint-3d55183b" direction="outbound" bytes_sent=76 bytes_recv=608 duration="3ms"
```

```bash
 kubectl -n envoy-gateway-system logs -l gateway.envoyproxy.io/owning-gateway-name=eg-waypoint --tail 1
Defaulted container "envoy" out of: envoy, shutdown-manager
{":authority":"backend:3000","bytes_received":0,"bytes_sent":466,"connection_termination_details":null,"downstream_local_address":"10.244.0.14:3000","downstream_remote_address":"10.244.0.12:34805","duration":1,"method":"GET","protocol":"HTTP/1.1","requested_server_name":null,"response_code":200,"response_code_details":"via_upstream","response_flags":"-","route_name":"httproute/envoy-gateway-system/backend/rule/0/match/0/backend","start_time":"2025-04-09T11:44:45.639Z","upstream_cluster":"httproute/envoy-gateway-system/backend/rule/0","upstream_host":"10.244.0.13:3000","upstream_local_address":"10.244.0.14:58822","upstream_transport_failure_reason":null,"user-agent":"curl/7.88.1","x-envoy-origin-path":"/","x-envoy-upstream-service-time":null,"x-forwarded-for":"10.244.0.12","x-request-id":"ab1e4da7-5f1d-45ea-bb4a-fa1ccd03b82b"}
```

## Try Envoy Gateway's Layer 7 routing features
All the Envoy Gateway's Layer 7 routing features are available, including the Gateway API features, Envoy Gateway's ClientTrafficPolicy, BackendTrafficPolicy, and SecurityPolicy.
You can follow the [Envoy Gateway's documentation](https://gateway.envoyproxy.io/docs/) to try them out.

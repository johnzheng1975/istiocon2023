This sample is about customize metrics, adding tenant_id. 


## Before you begin

1. Setup Istio in a Kubernetes cluster by following the instructions in the
   [Installation Guide](/docs/setup/getting-started/).

1. Deploy the [Bookinfo](https://istio.io/latest/docs/examples/bookinfo/) sample application.

1. Had better remove enovyfilter from ratelimit sample, to make logs clear.


## Customize Access Logs For tenant

1. Create envoyfilter to generate tenant_id from token.
   ```
   kubectl apply -f ./ef-sample-add-tenant_id.yaml
   ```

1. Edit serice mesh in istio configmap.
   ```
   $ kubectl edit configmap -n istio-system istio

   data:
     mesh: |-

       defaultConfig:
         ### Add metrics config begin
         extraStatTags:
          - tenant_id
         ### Add metrics config end  
         ... ...
		 
   ```

1. Wait some time, until istio configmap is updated in istiod and ingressgateway, or delete istio pods directly.
   ```
   $ kubectl delete pods -n istio-system --all
   ```

1. Create telemetry for new added metrics.
   ```
   $ kubectl apply -f telemetry.yaml
   
   #apiVersion: telemetry.istio.io/v1alpha1
   #kind: Telemetry
   #metadata:
   #  name: global-metrics
   #  namespace: istio-system
   #spec:
   #  metrics:
   #  - overrides:
   #    - tagOverrides:
   #        tenant_id:
   #          operation: UPSERT
   #          value: request.headers["tenant_id"] 
   #    providers:
   #    - name: prometheus
   ```

1. Send a http request with tenant_id in token. 
   ```
   $ curl a37a73937493a427db222f8deecd65ca-1310205472.us-east-1.elb.amazonaws.com/productpage -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwidGVuYW50X2lkIjoidGVuYW50MDAwMl9mcm9tdG9rZW4iLCJpYXQiOjE1MTYyMzkwMjJ9.9-mWgwS7r5_MEKfNPTnj6mTHAkAIel0Ri6dw0rCxn1U'
   ```
   
1. You can find metrics include `tenant_id`.
   ```
   $ kubectl exec "$(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}')" -c istio-proxy -- curl -sS 'localhost:15000/stats/prometheus' | grep istio_requests_total
   istio_requests_total{reporter="destination",source_workload="istio-ingressgateway",source_canonical_service="istio-ingressgateway",source_canonical_revision="latest",source_workload_namespace="istio-system",source_principal="spiffe://cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account",source_app="istio-ingressgateway",source_version="unknown",source_cluster="Kubernetes",destination_workload="productpage-v1",destination_workload_namespace="default",destination_principal="spiffe://cluster.local/ns/default/sa/bookinfo-productpage",destination_app="productpage",destination_version="v1",destination_service="productpage.default.svc.cluster.local",destination_canonical_service="productpage",destination_canonical_revision="v1",destination_service_name="productpage",destination_service_namespace="default",destination_cluster="Kubernetes",request_protocol="http",response_code="200",grpc_response_status="",response_flags="-",connection_security_policy="mutual_tls",tenant_id="tenant0002_fromtoken"} 1
   ```
 
## Other examples
- https://istio.io/latest/docs/tasks/observability/metrics/customize-metrics/


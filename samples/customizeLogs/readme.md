This sample is about customize logs, adding tenant_id, grpc_status. 


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
       ### Add logs config begin
       accessLogEncoding: JSON
       accessLogFile: /dev/stdout
       accessLogFormat: |
         {
              "authority": "%REQ(:AUTHORITY)%",
              "bytes_received": "%BYTES_RECEIVED%",
              "bytes_sent": "%BYTES_SENT%",
              "connection_termination_details": "%CONNECTION_TERMINATION_DETAILS%",
              "downstream_local_address": "%DOWNSTREAM_LOCAL_ADDRESS%",
              "downstream_remote_address": "%DOWNSTREAM_REMOTE_ADDRESS%",
              "duration": "%DURATION%",
              "method": "%REQ(:METHOD)%",
              "path": "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%",
              "protocol": "%PROTOCOL%",
              "request_id": "%REQ(X-REQUEST-ID)%",
              "requested_server_name": "%REQUESTED_SERVER_NAME%",
              "response_code": "%RESPONSE_CODE%",
              "response_code_details": "%RESPONSE_CODE_DETAILS%",
              "response_flags": "%RESPONSE_FLAGS%",
              "route_name": "%ROUTE_NAME%",
              "start_time": "%START_TIME%",
              "upstream_cluster": "%UPSTREAM_CLUSTER%",
              "upstream_host": "%UPSTREAM_HOST%",
              "upstream_local_address": "%UPSTREAM_LOCAL_ADDRESS%",
              "upstream_service_time": "%RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)%",
              "upstream_transport_failure_reason": "%UPSTREAM_TRANSPORT_FAILURE_REASON%",
              "user_agent": "%REQ(USER-AGENT)%",
              "x_forwarded_for": "%REQ(X-FORWARDED-FOR)%",
              "grpc_status_number": "%GRPC_STATUS_NUMBER%",
              "tenant_id": "%REQ(TENANT_ID)%"
          }
    ### Add logs config end
       defaultConfig:
         ... ...
   ```

1. Wait some time, until istio configmap is updated in istiod and ingressgateway, or delete istio pods directly.
   ```
   $ kubectl delete pods -n istio-system --all
   ```

1. Set ingressgateway pod lua log level to debug, open a window to view the log:
   ```
   $ kubectl exec -ti -n istio-system istio-ingressgateway-7485484874-gzz9s  -- curl -XPOST localhost:15000/logging?lua=debug
   $ kubectl logs -n istio-system istio-ingressgateway-7485484874-gzz9s -f
   ```

1. Send a http request with tenant_id in header, you can find access log include `grpc_status_number` and `tenant_id`.
   ```
   # Please put your istio-ingressgateway service loadbalancer domain before run curl.
   $ curl a37a73937493a427db222f8deecd65ca-1310205472.us-east-1.elb.amazonaws.com/productpage -H "TENANT_ID: tenant-00001"

   $ Log found in istio-ingressgateway-7485484874-gzz9s pod:
   {
       "authority": "a37a73937493a427db222f8deecd65ca-1310205472.us-east-1.elb.amazonaws.com",
       "bytes_received": 0,
       "bytes_sent": 5290,
       "connection_termination_details": null,
       "downstream_local_address": "192.168.48.162:8080",
       "downstream_remote_address": "192.168.52.62:2960",
       "duration": 48,
       "grpc_status_number": 2,                                              ##### New customized field
       "method": "GET",
       "path": "/productpage",
       "protocol": "HTTP/1.1",
       "request_id": "27842d9a-1df6-43b9-94ac-21b98f168159",
       "requested_server_name": null,
       "response_code": 200,
       "response_code_details": "via_upstream",
       "response_flags": "-",
       "route_name": null,
       "start_time": "2023-09-21T07:29:37.776Z",
       "tenant_id": "tenant-00001",                                          ###### New customized field
       "upstream_cluster": "outbound|9080||productpage.default.svc.cluster.local",
       "upstream_host": "192.168.30.230:9080",
       "upstream_local_address": "192.168.48.162:39142",
       "upstream_service_time": "47",
       "upstream_transport_failure_reason": null,
       "user_agent": "curl/7.68.0",
       "x_forwarded_for": "192.168.52.62"
   }
   ```

1. Send a http request with tenant_id in token, you can find access log include `grpc_status_number` and `tenant_id`.
   ```
   $ curl a37a73937493a427db222f8deecd65ca-1310205472.us-east-1.elb.amazonaws.com/productpage -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwidGVuYW50X2lkIjoidGVuYW50MDAwMl9mcm9tdG9rZW4iLCJpYXQiOjE1MTYyMzkwMjJ9.9-mWgwS7r5_MEKfNPTnj6mTHAkAIel0Ri6dw0rCxn1U'

   $ Log found in istio-ingressgateway-7485484874-gzz9s pod:
   {
       "authority": "a37a73937493a427db222f8deecd65ca-1310205472.us-east-1.elb.amazonaws.com",
       "bytes_received": 0,
       "bytes_sent": 5294,
       "connection_termination_details": null,
       "downstream_local_address": "192.168.48.162:8080",
       "downstream_remote_address": "192.168.52.62:18001",
       "duration": 57,
       "grpc_status_number": 2,                                              ###### New customized field
       "method": "GET",
       "path": "/productpage",
       "protocol": "HTTP/1.1",
       "request_id": "d91d4a1f-b07c-4a58-82e4-ed9016da0b5e",
       "requested_server_name": null,
       "response_code": 200,
       "response_code_details": "via_upstream",
       "response_flags": "-",
       "route_name": null,
       "start_time": "2023-09-21T07:26:16.820Z",
       "tenant_id": "tenant0002_fromtoken",                                  ###### New customized field
       "upstream_cluster": "outbound|9080||productpage.default.svc.cluster.local",
       "upstream_host": "192.168.30.230:9080",
       "upstream_local_address": "192.168.48.162:39142",
       "upstream_service_time": "55",
       "upstream_transport_failure_reason": null,
       "user_agent": "curl/7.68.0",
       "x_forwarded_for": "192.168.52.62"
   }
   ```



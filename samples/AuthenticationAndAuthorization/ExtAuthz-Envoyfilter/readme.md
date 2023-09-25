This sample is about ext-authz envoyfilter usage. 

## Before you begin

Before you begin this task, do the following:

* Read the [Istio authorization concepts](https://istio.io/latest/docs/concepts/security/#authorization).

* Follow the [Istio installation guide](https://istio.io/latest/docs/setup/install/istioctl/) to install Istio.

* Deploy test workloads:

    This task uses two workloads, `httpbin` and `sleep`, both deployed in namespace `foo`.
    Both workloads run with an Envoy proxy sidecar. Deploy the `foo` namespace
    and workloads with the following command:

    ```
    $ kubectl create ns foo
    $ kubectl label ns foo istio-injection=enabled
    $ kubectl apply -f samples/httpbin/httpbin.yaml -n foo   # under istio-1.18.2 install folder
    $ kubectl apply -f samples/sleep/sleep.yaml -n foo   # under istio-1.18.2 install folder
    ```

* Verify that `sleep` can access `httpbin` with the following command:

    ```
    $ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl http://httpbin.foo:8000/ip -s -o /dev/null -w "%{http_code}\n"
    200
    ```

If you don’t see the expected output as you follow the task, retry after a few seconds.
Caching and propagation overhead can cause some delay.

## Deploy the external authorizer

First, you need to deploy the external authorizer. For this, you will simply deploy the sample external authorizer in a standalone pod in the mesh.

1. Run the following command to deploy the sample external authorizer:

    ```
    $ kubectl apply -n foo -f ./samples/extauthz/ext-authz.yaml  # under istio-1.18.2 install folder
    service/ext-authz created
    deployment.apps/ext-authz created
    ```

1. Verify the sample external authorizer is up and running:

    ```
    $ kubectl logs "$(kubectl get pod -l app=ext-authz -n foo -o jsonpath={.items..metadata.name})" -n foo -c ext-authz
    2021/01/07 22:55:47 Starting HTTP server at [::]:8000
    2021/01/07 22:55:47 Starting gRPC server at [::]:9000
    ```

## grpcservice envoyfilter
### Deploy the ext-authz envoyfilter (grpcservice)

First, you need to deploy ext-authz envoyfilter.

1. Run the command to deploy:

    ```
    ~/istiocon2023/samples/AuthenticationAndAuthorization/ExtAuthz-Envoyfilter$  kubectl create -f ./envoyfilter-ext-authz-grpcservice.yaml
      envoyfilter.networking.istio.io/httpbin created
      envoyfilter.networking.istio.io/ext-authz-cluster-patch created
    ```
2. Wait some seconds, before testing. Make sure envoyfitler is synced to istio-proxy.
   
### Test
Here is test result:

1. Verify a request to path `/post` with header `x-ext-authz: deny` is denied by the sample `ext_authz` server:

    ```
    $ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl -XPOST "http://httpbin.foo:8000/post" -H "x-ext-authz: deny" -H "key2: value2" --header 'Content-Type: text/plain' --data '1234567890111111'
    denied by ext_authz for not found header `x-ext-authz: allow` in the request
    ```

1. Verify a request to path `/headers` with header `x-ext-authz: allow` is allowed by the sample `ext_authz` server:
    ```
    $ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl -XPOST "http://httpbin.foo:8000/post" -H "x-ext-authz: allow" -H "key2: value2" --header 'Content-Type: text/plain' --data '1234567890111111'
    {
      "args": {},
      "data": "1234567890111111",
      "files": {},
      "form": {},
      "headers": {
        "Accept": "*/*",
        "Content-Length": "16",
        "Content-Type": "text/plain",
        "Host": "httpbin.foo:8000",
        "Key2": "value2",
        "User-Agent": "curl/8.3.0",
        "X-B3-Parentspanid": "4b07af7914efdf50",
        "X-B3-Sampled": "0",
        "X-B3-Spanid": "5e6a9e6611cf2e20",
        "X-B3-Traceid": "250e3ae4875378054b07af7914efdf50",
        "X-Envoy-Attempt-Count": "1",
        "X-Ext-Authz": "allow",
        "X-Ext-Authz-Additional-Header-Override": "grpc-additional-header-override-value",
        "X-Ext-Authz-Check-Received": "source:{address:{socket_address:{address:\"192.168.51.97\" port_value:55106}} principal:\"spiffe://cluster.local/ns/foo/sa/sleep\"} destination:{address:{socket_address:{address:\"192.168.42.234\" port_value:80}} principal:\"spiffe://cluster.local/ns/foo/sa/httpbin\"} request:{time:{seconds:1694935150 nanos:259058000} http:{id:\"42739569984879099\" method:\"POST\" headers:{key:\":authority\" value:\"httpbin.foo:8000\"} headers:{key:\":method\" value:\"POST\"} headers:{key:\":path\" value:\"/post\"} headers:{key:\":scheme\" value:\"http\"} headers:{key:\"accept\" value:\"*/*\"} headers:{key:\"content-length\" value:\"16\"} headers:{key:\"content-type\" value:\"text/plain\"} headers:{key:\"key2\" value:\"value2\"} headers:{key:\"user-agent\" value:\"curl/8.3.0\"} headers:{key:\"x-b3-sampled\" value:\"0\"} headers:{key:\"x-b3-spanid\" value:\"4b07af7914efdf50\"} headers:{key:\"x-b3-traceid\" value:\"250e3ae4875378054b07af7914efdf50\"} headers:{key:\"x-envoy-attempt-count\" value:\"1\"} headers:{key:\"x-envoy-auth-partial-body\" value:\"false\"} headers:{key:\"x-ext-authz\" value:\"allow\"} headers:{key:\"x-forwarded-client-cert\" value:\"By=spiffe://cluster.local/ns/foo/sa/httpbin;Hash=ea75feef24f7131ed1f1bcc2c52846f965df4ae4937d764f9f48ed439a75a748;Subject=\\\"\\\";URI=spiffe://cluster.local/ns/foo/sa/sleep\"} headers:{key:\"x-forwarded-proto\" value:\"http\"} headers:{key:\"x-request-id\" value:\"c86d8c0f-5afd-4655-90e3-08c19c474454\"} path:\"/post\" host:\"httpbin.foo:8000\" scheme:\"http\" size:16 protocol:\"HTTP/1.1\" body:\"1234567890111111\"}} metadata_context:{}",
        "X-Ext-Authz-Check-Result": "allowed",
        "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/foo/sa/httpbin;Hash=ea75feef24f7131ed1f1bcc2c52846f965df4ae4937d764f9f48ed439a75a748;Subject=\"\";URI=spiffe://cluster.local/ns/foo/sa/sleep"
      },
      "json": 1234567890111111,
      "origin": "127.0.0.6",
      "url": "http://httpbin.foo:8000/post"
    }
    ```

### Clean up
    ```
    $ kubectl delete -f ./envoyfilter-ext-authz-grpcservice.yaml
    envoyfilter.networking.istio.io "httpbin" deleted
    envoyfilter.networking.istio.io "ext-authz-cluster-patch" deleted
    ```

## httpservice envoyfilter
### Deploy the ext-authz envoyfilter (httpservice)

First, you need to deploy ext-authz envoyfilter.

1. Run the command to deploy:

    ```
    ~/istiocon2023/samples/AuthenticationAndAuthorization/ExtAuthz-Envoyfilter$  kubectl create -f ./envoyfilter-ext-authz-httpservice.yaml
    ```

### Test
Here is test result:

1. Verify a request to path `/post` with header `x-ext-authz: deny` is denied by the sample `ext_authz` server:

    ```
    $ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl -XPOST "http://httpbin.foo:8000/post" -H "x-ext-authz: deny" -H "key2: value2" --header 'Content-Type: text/plain' --data '1234567890111111'
    denied by ext_authz for not found header `x-ext-authz: allow` in the request
    ```

1. Verify a request to path `/headers` with header `x-ext-authz: allow` is allowed by the sample `ext_authz` server:
    ```
    $ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl -XPOST "http://httpbin.foo:8000/post" -H "x-ext-authz: allow" -H "key2: value2" --header 'Content-Type: text/plain' --data '1234567890111111'
    {
      "args": {},
      "data": "1234567890111111",
      "files": {},
      "form": {},
      "headers": {
        "Accept": "*/*",
        "Content-Length": "16",
        "Content-Type": "text/plain",
        "Host": "httpbin.foo:8000",
        "Key2": "value2",
        "User-Agent": "curl/8.3.0",
        "X-B3-Parentspanid": "53af88335d8fd629",
        "X-B3-Sampled": "0",
        "X-B3-Spanid": "919bd65cad8a6c46",
        "X-B3-Traceid": "1ad87712ed05cce053af88335d8fd629",
        "X-Envoy-Attempt-Count": "1",
        "X-Ext-Authz": "allow",
        "X-Ext-Authz-Check-Received": "POST httpbin.foo:8000/post, headers: map[Content-Length:[0] X-B3-Parentspanid:[ee9025124c57e4f8] X-B3-Sampled:[0] X-B3-Spanid:[601a5da7857a2948] X-B3-Traceid:[1ad87712ed05cce053af88335d8fd629] X-Envoy-Expected-Rq-Timeout-Ms:[10000] X-Envoy-Internal:[true] X-Ext-Authz:[allow] X-Forwarded-Client-Cert:[By=spiffe://cluster.local/ns/foo/sa/default;Hash=624e241c728de3629ba18b36893e5c660f59fa480c45af049e674088dbaa5b28;Subject=\"\";URI=spiffe://cluster.local/ns/foo/sa/httpbin] X-Forwarded-For:[192.168.5.36] X-Forwarded-Proto:[https] X-Request-Id:[61d5803b-77af-4650-a4b9-6b28b26704a2]], body: []",
        "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/foo/sa/httpbin;Hash=8de6d8ab8563b4495e9635886014ced414ba6b0d66f0e32966e7ca919106183c;Subject=\"\";URI=spiffe://cluster.local/ns/foo/sa/sleep"
      },
      "json": 1234567890111111,
      "origin": "127.0.0.6",
      "url": "http://httpbin.foo:8000/post"
    }
    ```

### Clean up
    ```
    $ kubectl delete -f ./envoyfilter-ext-authz-httpservice.yaml

    ```

## Enabled Debug And Test again
- In order to better understand how it works, suggest you [enable debug](https://github.com/johnzheng1975/istiocon2023/tree/main/samples/AuthenticationAndAuthorization#enable-debug-before-go-through-samples) for below `Advanaced usage` try.

## Advanced usage

Here are some thoughts/ tips for ext-authz envoyfilter:

1. For technical details for envoy ext-authz envoyfilter, you can refer to:
   https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/ext_authz/v3/ext_authz.proto#envoy-v3-api-msg-extensions-filters-http-ext-authz-v3-extauthz

2. You can use grpc_service or http_service for external authorizer.

3. External authorizer can return allow or deny based on request headers. So http headers need be forwarded to External authorizer.
   - For http_service, for headers need be forwareded, please them in allowed_headers. Otherwise, will not be forwarded.
   - For grpc_service, all headers will be forwarded if allowed_headers is not defined. However, if allowed_headers is defined, the other headers will not be forwarded.

4. External authorizer can return allow or deny based on request body. Below works for both grpc_service and http_service

   ```
        typed_config:
          with_request_body:
            max_request_bytes: 1024
            allow_partial_message: true
            pack_as_bytes: false
   ```

   Suggest set `allow_partial_message` to true. Otherwise, Envoy will return HTTP 413 if body size more than max_request_bytes.

5. For http_service, it supports AuthorizationResponse definition, like

   ```
     "allowed_upstream_headers": {...},
     "allowed_upstream_headers_to_append": {...},
     "allowed_client_headers": {...},
     "allowed_client_headers_on_success": {...},
     "dynamic_metadata_from_headers": {...}
   ```

   It can remove some unnecessary big headers like X-Ext-Authz-Check-Received, and improve performance.

6. For grpc_service, it does not support AuthorizationResponse definition. 



## Envoyfilter sample

### Simple grpc_service
- Body will be forwarded to external authorizer.
- All headers will be forwarded to external authorizer.
```
$ cat envoyfilter-ext-authz-simple.yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: httpbin
  namespace: foo
spec:
  workloadSelector:
    labels:
      app: httpbin
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
      listener:
        filterChain:
          filter:
            name: "envoy.filters.network.http_connection_manager"
            subFilter:
              name: "envoy.filters.http.router"
    patch:
      operation: INSERT_BEFORE
      value:
        name: "envoy.ext_authz"
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz
          failure_mode_allow: false
          transport_api_version: V3
          grpc_service:
            envoy_grpc:
              cluster_name: patched.extauthz.foo.svc.cluster.local
          with_request_body:
            max_request_bytes: 1024
            allow_partial_message: true
            pack_as_bytes: false
---
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: ext-authz-cluster-patch
  namespace: foo
spec:
  workloadSelector:
    labels:
      app: httpbin
  configPatches:
  - applyTo: CLUSTER
    match:
      cluster:
        service: ext-authz.foo.svc.cluster.local
        portNumber: 9000
    patch:
      operation: MERGE
      value:
        name: "patched.extauthz.foo.svc.cluster.local"

```

### http_service
- Body will be forwarded to external authorizer.
- Headers defined in `allowed_headers` will be forwarded to external authorizer.
- Headers defined in `allowed_upstream_headers` will be forwarded to httpbin from external authorizer.
```
$ cat envoyfilter-ext-authz-httpservice.yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: httpbin
  namespace: foo
spec:
  workloadSelector:
    labels:
      app: httpbin
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
      listener:
        filterChain:
          filter:
            name: "envoy.filters.network.http_connection_manager"
            subFilter:
              name: "envoy.filters.http.router"
    patch:
      operation: INSERT_BEFORE
      value:
        name: "envoy.ext_authz"
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz
          failure_mode_allow: false
          allowed_headers:
            patterns:
            - exact: baz
            - exact: x-ext-authz
          transport_api_version: V3
          http_service:
            server_uri:
              cluster: outbound|8000||ext-authz.foo.svc.cluster.local
              timeout: 10s
              uri: http://ext-authz.foo.svc.cluster.local:8000/
            authorization_response:
              allowed_upstream_headers:
                patterns:
                - exact: "abc"
                - exact: "x-ext-authz-check-received"

```


## Clean up

1. Remove the namespace `foo` from your configuration:

    ```
    $ kubectl delete namespace foo
    ```


 
 

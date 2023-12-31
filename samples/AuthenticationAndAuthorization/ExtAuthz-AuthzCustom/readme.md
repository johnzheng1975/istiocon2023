This sample is about External Authorization usage. 

## Quick Start
* Please refer to: https://istio.io/latest/docs/tasks/security/authorization/authz-custom/

## Go through this example 
1. Edit istio mesh configmap.

  ```
  # kubectl edit configmap -n istio-system istio
    
    mesh: |-
      extensionProviders:
      - name: "sample-ext-authz-grpc"
        envoyExtAuthzGrpc:
          service: "ext-authz.foo.svc.cluster.local"
          port: "9000"
          includeRequestBodyInCheck:
            maxRequestBytes: 1000000
            allowPartialMessage: false
            packAsBytes: false
      - name: "sample-ext-authz-http"
        envoyExtAuthzHttp:
          service: "ext-authz.foo.svc.cluster.local"
          port: "8000"
          includeRequestHeadersInCheck: ["x-ext-authz", "baz"]
          headersToUpstreamOnAllow: ["x-ext-authz-check-received"]
          includeRequestBodyInCheck:
            maxRequestBytes: 1000000
            allowPartialMessage: true
            packAsBytes: false
  ```

2. Apply authoriationpolicy-grpc.yaml

  ```
  kubectl create -f authoriationpolicy-grpc.yaml -n foo
  ```

3. Test (From result, you can find `POST httpbin.foo:9000` which is ext-authz grpc port)
  ```
  # deny case 
  $ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl -XPOST "http://httpbin.foo:8000/post" -H "x-ext-authz: deny" -H "key2: value2" --header 'Content-Type: text/plain' --data '1234567890111111'
  
  # allow case
  $ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl -XPOST "http://httpbin.foo:8000/post" -H "x-ext-authz: allow" -H "key2: value2" --header 'Content-Type: text/plain' --data '1234567890111111'
  ```

4. Use authoriationpolicy-http.yaml

  ```
  kubectl delete -f authoriationpolicy-grpc.yaml -n foo
  kubectl create -f authoriationpolicy-http.yaml -n foo
  ```

5. Test (From result, you can find `POST httpbin.foo:8000` which is ext-authz http port)
  ```
  # deny case 
  $ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl -XPOST "http://httpbin.foo:8000/post" -H "x-ext-authz: deny" -H "key2: value2" --header 'Content-Type: text/plain' --data '1234567890111111'
  
  # allow case
  $ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl -XPOST "http://httpbin.foo:8000/post" -H "x-ext-authz: allow" -H "key2: value2" --header 'Content-Type: text/plain' --data '1234567890111111'
  ```

## Best Practice
Here are some thoughts/ tips for ext-authz envoyfilter:

1. Comparing with ext-authz envoyfilter, it can support ext-authz flow conditionally, enable/disable for a specific route based on path/host/IP/etc. 

2. This feature is similar with ext-authz envoyfilter, also based in ext-authz envoyfilter. But more convenience.

3. The main configuration is in istio configmap, like below
   ```
   # There are many comments below.
   # For Useful, this means you can set
   # For Not Useful, means does not support this.
   mesh: |-
     extensionProviders:
     - name: "sample-ext-authz-grpc"
       envoyExtAuthzGrpc:
         service: "ext-authz.foo.svc.cluster.local"
         port: "9000"
         includeRequestHeadersInCheck: ["x-ext-authz"]  # Not useful, no matter set or not set.
         headersToUpstreamOnAllow: ["abc"]   # Not useful, no matter set or not set.
         includeRequestBodyInCheck:  # Useful.
           maxRequestBytes: 1000000
           allowPartialMessage: false
           packAsBytes: false
     - name: "sample-ext-authz-http"
       envoyExtAuthzHttp:
         service: "ext-authz.foo.svc.cluster.local"
         port: "8000"
         includeRequestHeadersInCheck: ["x-ext-authz", "baz"]   # Useful.
         headersToUpstreamOnAllow: ["x-ext-authz-check-received"]   # Useful.
         includeRequestBodyInCheck:   # Useful.
           maxRequestBytes: 1000000
           allowPartialMessage: true
           packAsBytes: false
   ```
	
4. Details configuration for ExtensionProvider EnvoyExternalAuthorization, can refer to:
   - https://preliminary.istio.io/latest/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-ExtensionProvider-EnvoyExternalAuthorizationHttpProvider
   - https://preliminary.istio.io/latest/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-ExtensionProvider-EnvoyExternalAuthorizationGrpcProvider
 
5. It supports envoyExtAuthzGrpc or envoyExtAuthzHttp.

6. For envoyExtAuthzHttp, you can define 
   - includeRequestHeadersInCheck (Headers can be forwarded to authz server. Same as allowed_headers in [ext-authz envoyfilter](https://github.com/johnzheng1975/istiocon2023/tree/main/samples/AuthenticationAndAuthorization/ExtAuthz-Envoyfilter).)
   - headersToUpstreamOnAllow (The headers from ext-authz service, will be added or overridden in the original request and forwarded to the upstream. Same as allowed_upstream_headers in [ext-authz envoyfilter](https://github.com/johnzheng1975/istiocon2023/tree/main/samples/AuthenticationAndAuthorization/ExtAuthz-Envoyfilter).)
   
7. For envoyExtAuthzGrpc, you cannot define includeRequestHeadersInCheck or headersToUpstreamOnAllow. All headers will be forwarded. This may impact the performance.

8. External authorizer can return allow or deny based on request body, if you define includeRequestBodyInCheck below. it works for both grpc_service and http_service. 

   ```
         includeRequestBodyInCheck:  # Useful.
           maxRequestBytes: 1000000
           allowPartialMessage: false
           packAsBytes: false
   ```

   Similar with with_request_body in envoyfilter.

9. For upper mesh configuration, the envoyfilter is generated in istio-proxy as below finally:
   ```
   $ kubectl exec -ti "$(kubectl get pod -l app=httpbin -n foo -o jsonpath={.items..metadata.name})"  -n foo -c istio-proxy -- curl localhost:15000/config_dump
   
   $ For the upper envoyExtAuthzGrpc configure, will generate final envoyfilter like:
             {
              "name": "envoy.filters.http.ext_authz",
              "typed_config": {
               "@type": "type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz",
               "grpc_service": {
                "envoy_grpc": {
                 "cluster_name": "outbound|9000||ext-authz.foo.svc.cluster.local",
                 "authority": "outbound_.9000_._.ext-authz.foo.svc.cluster.local"
                },
                "timeout": "600s"
               },
               "with_request_body": {
                "max_request_bytes": 1000000
               },
               "transport_api_version": "V3",
               "filter_enabled_metadata": {
                "filter": "envoy.filters.http.rbac",
                "path": [
                 {
                  "key": "istio_ext_authz_shadow_effective_policy_id"
                 }
                ],
                "value": {
                 "string_match": {
                  "prefix": "istio-ext-authz"
                 }
                }
               }
              }
             },


   $ For the upper envoyExtAuthzHttp configure, will generate final envoyfilter like:
             {
              "name": "envoy.filters.http.ext_authz",
              "typed_config": {
               "@type": "type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz",
               "http_service": {
                "server_uri": {
                 "uri": "http://ext-authz.foo.svc.cluster.local",
                 "cluster": "outbound|8000||ext-authz.foo.svc.cluster.local",
                 "timeout": "600s"
                },
                "authorization_response": {
                 "allowed_upstream_headers": {
                  "patterns": [
                   {
                    "exact": "x-ext-authz-check-received",
                    "ignore_case": true
                   }
                  ]
                 }
                }
               },
               "with_request_body": {
                "max_request_bytes": 1000000,
                "allow_partial_message": true
               },
               "transport_api_version": "V3",
               "filter_enabled_metadata": {
                "filter": "envoy.filters.http.rbac",
                "path": [
                 {
                  "key": "istio_ext_authz_shadow_effective_policy_id"
                 }
                ],
                "value": {
                 "string_match": {
                  "prefix": "istio-ext-authz"
                 }
                }
               },
               "allowed_headers": {
                "patterns": [
                 {
                  "exact": "x-ext-authz",
                  "ignore_case": true
                 },
                 {
                  "exact": "baz",
                  "ignore_case": true
                 }
                ]
               }
              }
             },
   ```
 

## Clean up

1. Remove the namespace `foo` from your configuration:

    ```
    $ kubectl delete namespace foo
    ```

## Introduction from Istio developer
- Video 
  * https://www.youtube.com/watch?v=BSmckzCAuk8
- PDF
  * https://events.istio.io/istiocon-2021/slides/g8p-BetterExternalAuthorization-YangminZhu.pdf
- Design docs
  * https://docs.google.com/document/d/1V4mCQCw7mlGp0zSQQXYoBdbKMDnkPOjeyUb85U07iSI

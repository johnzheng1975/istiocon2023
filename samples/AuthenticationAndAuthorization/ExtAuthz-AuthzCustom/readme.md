This sample is about External Authorization usage. 

## Quick Start
* Please refer to: https://istio.io/latest/docs/tasks/security/authorization/authz-custom/

## Go through this example, through changing the istio configmap.
```
# kubectl edit configmap -n istio-system istio
  
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

## Introduction from its developer
- Video 
  * https://www.youtube.com/watch?v=BSmckzCAuk8
- PDF
  * https://events.istio.io/istiocon-2021/slides/g8p-BetterExternalAuthorization-YangminZhu.pdf
- Design docs
  * https://docs.google.com/document/d/1V4mCQCw7mlGp0zSQQXYoBdbKMDnkPOjeyUb85U07iSI
 

## Best Practice
Here are some thoughts/ tips for ext-authz envoyfilter:

1. Comparing with ext-authz envoyfilter, it can support ext-authz flow conditionally, enable/disable for a specific route based on path/host/IP/etc. 

2. This feature is similar or even based on ext-authz envoyfilter. But more convenience.

3. The main configuration is in istio configmap, like below
   ```
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
   - includeRequestHeadersInCheck (It can be forwarded to authz server. Same as allowed_headers in envoyfilter.)
   - headersToUpstreamOnAllow (It is from the authorization service, will be added or overridden in the original request and forwarded to the upstream. Same as allowed_upstream_headers in envoyfilter.)
   
7. For envoyExtAuthzGrpc, you cannot define includeRequestHeadersInCheck or headersToUpstreamOnAllow. All headers will be included. This may impact the performance.

8. External authorizer can return allow or deny based on request body, if you define includeRequestBodyInCheck below. it works for both grpc_service and http_service. 

   ```
         includeRequestBodyInCheck:  # Useful.
           maxRequestBytes: 1000000
           allowPartialMessage: false
           packAsBytes: false
   ```

   Similar with with_request_body in envoyfilter.

9. For upper mesh configuration, the envoyfilter is generated as below:
   ```
   $ kubectl exec -ti "$(kubectl get pod -l app=httpbin -n foo -o jsonpath={.items..metadata.name})"  -n foo -c istio-proxy -- curl localhost:15000/config_dump
   
   $ For the upper envoyExtAuthzGrpc configure, final envoyfilter like:
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


   $ For the upper envoyExtAuthzHttp configure, final envoyfilter like:
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
 
10. How to use this istio mesh configure is simple: just define an AuthorizationPolicy like below.
   ```
   apiVersion: security.istio.io/v1
   kind: AuthorizationPolicy
   metadata:
     name: ext-authz
   spec:
     selector:
       matchLabels:
         app: httpbin
     action: CUSTOM
     provider:
       # The provider name must match the extension provider defined in the mesh config.
       # You can also replace this with sample-ext-authz-http to test the other external authorizer definition.
       name: sample-ext-authz-grpc
     rules:
     # The rules specify when to trigger the external authorizer.
     - to:
       - operation:
           paths: ["/headers"]
   ```

11. Test 
   Suggest you [enable debug log](../readme.md), during your testing.
   ```
   # deny case 
   $ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl -XPOST "http://httpbin.foo:8000/post" -H "x-ext-authz: deny" -H "key2: value2" --header 'Content-Type: text/plain' --data '1234567890111111'
  
   # allow case
   $ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl -XPOST "http://httpbin.foo:8000/post" -H "x-ext-authz: allow" -H "key2: value2" --header 'Content-Type: text/plain' --data '1234567890111111'
   ```
   
   Suggest you [enable debug log](../readme.md), during your testing.
   

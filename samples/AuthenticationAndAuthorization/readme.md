## Introduction
- For ext-auth envoyfilter / External Authorization, current samples is not much on internet.
- Here created samples for them, it not only includes simple "Get started",but also includes our experience and lessons learned.

## Enable Debug Before Go Through Samples
- Suggest you apply debug envoyfilter, to show all http request/response headers/body for ext-authz pod. This can make you better understand, about how "ext-auth envoyfilter" and "External Authorization" works.
- Here are commond, please run it after namespaces & app created.
  ```
  # Apply debug envoyfilter, which will print out all request/response headers/body, no matter it is http or grpc.
  kubectl apply -f ./envoyfilter-show_RequestsResponse.yaml
  
  # After you create ext-authz pod in sample, you can open another window to view log:  
  # Show debug log for ext-authz 
  kubectl exec -ti -n foo "$(kubectl get pod -l app=ext-authz -n foo -o jsonpath={.items..metadata.name})"  -c istio-proxy -- curl -XPOST localhost:15000/logging?lua=debug
  kubectl logs -n foo "$(kubectl get pod -l app=ext-authz -n foo -o jsonpath={.items..metadata.name})"  -c istio-proxy  -f
  ```

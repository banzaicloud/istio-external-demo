## Demo for restricting access to external services in Istio

> For more info on this topic, read our blog post: https://banzaicloud.com/blog/istio-external-demo

Problems that this demo aims to solve:

- Deny access for all external services by default
- Allow to call external services for each applications separately (even in the same namespace)
- Traffic to external services should always go through egress gateways so that requests can be monitored

### Demo architecture

pic

We will have the following two namespaces: `test-app` and `external`.
The following two applications will be in the `test-app` namespace: `app-a` and `app-b`.
In the `external` namespace there will be three egress gateways for three external services: `github.com`, `google.com` and `httpbin.org`.
We allow the following connections:

- `app-a` --> `github.com`
- `app-a` --> `httpbin.org`
- `app-b` --> `google.com`

All other external services should be denied.

### Try demo

1. Create a Kubernetes cluster

1. Point KUBECONFIG for your cluster

1. `backyards install -a`

   In this demo, we install Istio with [Backyards](https://banzaicloud.com/products/backyards/), but the resources can be used on any Istio mesh.
   > Note: Banzai Cloud [Backyards](https://banzaicloud.com/products/backyards/) is our service-mesh product based on Istio. You can freely test and evaluate it in non-production environments. [Contact us](https://banzaicloud.com/contact/) if you'd like to use Backyards in production.

1. `kubectl create ns test-app`

   We will place our sample applications in the `test-app` namespace.

1. `backyards sidecar-proxy auto-inject on test-app`

   We turn on sidecar-injection for the `test-app` namespace with the above command (could be labeled manually as well).

1. `kubectl create ns external`

   This is where the external services will be placed, which will be be only accessible for specific applications.

1. `kubectl apply --recursive -f resources/`

   We create all the resources for this demo with the above command to the `test-app` and `external` namespaces.

    Now, try our configurations:
1. `kubectl exec -n=test-app -ti <APP-A-POD-NAME> -- curl -s -o /dev/null -w "%{http_code}" httpbin.org`

    Should be 200.

1. `kubectl exec -n=test-app -ti <APP-A-POD-NAME> -- curl -s -o /dev/null -w "%{http_code}" github.com`

    Should be 200.

1. `kubectl exec -n=test-app -ti <APP-A-POD-NAME> -- curl -s -o /dev/null -w "%{http_code}" google.com`

    Should be 502, as it is not allowed explicitly.

1. `kubectl exec -n=test-app -ti <APP-B-POD-NAME> -- curl -s -o /dev/null -w "%{http_code}" google.com`

    Should be 200.

1. `kubectl exec -n=test-app -ti <APP-B-POD-NAME> -- curl -s -o /dev/null -w "%{http_code}" httpbin.org`

    Should be 502, as it is not allowed explicitly.

1. `kubectl exec -n=test-app -ti <APP-B-POD-NAME> -- curl -s -o /dev/null -w "%{http_code}" github.com`

    Should be 502, as it is not allowed explicitly.

1. `backyards dashboard`

   This last step is optional, but if you create such traffic as mentioned above and checkout the `test-app` and `external` namespaces on the `TOPOLOGY` view you should see something like this:
   
   ![](/pics/by-topology.png)

### Cleanup

1. `kubectl delete --recursive -f resources/`
1. `kubectl delete ns test-app`
1. `kubectl delete ns external`

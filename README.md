# Restricting access to external services in Istio

> This example setup was primarily prepared for this [blog post](https://banzaicloud.com/blog/istio-external-demo), where you can read more.

## Goals with the demo

- Deny access towards all external services by default (from a particular application namespace)
- Explicitly whitelist which applications are allowed to call which external services (even if applications are in the same namespace)
- Monitor all calls towards external services by making sure traffic goes through egress gateways
- Support both HTTP and HTTPS calls

## Demo architecture

pic

We will have the following two namespaces: `test-app` and `external`.
The following two applications will be in the `test-app` namespace: `app-a` and `app-b`.
In the `external` namespace, there will be three egress gateways for three external services: `github.com`, `google.com` and `httpbin.org`.
We will allow the following connections:

- `app-a` --> `http(s)://github.com`
- `app-a` --> `http(s)://httpbin.org`
- `app-b` --> `http(s)://google.com`

All other external services should be denied from the `test-app` namespace.

## Try it out!

### Setup

1. Create a Kubernetes cluster

   > If you need a hand with that, you can create a cluster with the free version of [Banzai Cloud's Pipeline platform](https://beta.banzaicloud.io/).

1. Point `KUBECONFIG` for your cluster

1. Run:
   ```bash
   curl https://getbackyards.sh | sh && backyards install -a
   ```

   In this demo, we install Istio with [Backyards](https://banzaicloud.com/products/backyards/), but the resources in this repo can be used on any Istio mesh.
   > Note: [Backyards](https://banzaicloud.com/products/backyards/) is Banzai Cloud's service-mesh product based on Istio. You can freely test and evaluate it in non-production environments. [Contact us](https://banzaicloud.com/contact/) if you'd like to use Backyards in production.

   This step usually takes a few minutes, it installs Istio and Backyards on your cluster.

1. Run:
   ```bash
   kubectl create ns test-app
   ```

   We will place our sample applications in the `test-app` namespace.

1. Run:
   ```bash
   backyards sidecar-proxy auto-inject on test-app
   ```

   We turn on sidecar-injection for the `test-app` namespace with the above command.
   (Optionally, the namespace could be labeled manually as well.)
   
1. Run:
   ```bash
   kubectl get ns test-app --show-labels
   ```
   
   We run this command to make sure that the `test-app` namespace has the necessary `istio-injection=enabled` label on. (If not there yet, wait a few seconds and check again.)

1. Run:
   ```bash
   kubectl create ns external
   ```

   This is where the external services will be placed, which will be be only accessible for specific applications.

1. Run:
   ```bash
   kubectl apply --recursive -f https://github.com/banzaicloud/istio-external-demo/tree/master/resources/
   ```

   We create all the resources for this demo with the above command to the `test-app` and `external` namespaces.

### Explore

1. Run:
   ```bash
   APP_A_POD_NAME=$(kubectl get pods -n test-app -l k8s-app=app-a -o=jsonpath='{.items[0].metadata.name}')
   APP_B_POD_NAME=$(kubectl get pods -n test-app -l k8s-app=app-b -o=jsonpath='{.items[0].metadata.name}')
   ```
   
   We save the application pod's names for easier access.

1. Run:
   ```bash
   kubectl exec -n=test-app -ti $APP_A_POD_NAME -- curl -Ls -o /dev/null -w "%{http_code}" httpbin.org
   ```
   
    Should be 200, as we specifically whitelisted `app-a` --> `http(s)://httpbin.org`.

1. Run:
   ```bash
   kubectl exec -n=test-app -ti $APP_A_POD_NAME -- curl -Ls -o /dev/null -w "%{http_code}" github.com
   ```
   
    Should be 200, as we specifically whitelisted `app-a` --> `http(s)://github.com`.

1. Run:
   ```bash
   kubectl exec -n=test-app -ti $APP_A_POD_NAME -- curl -Ls -o /dev/null -w "%{http_code}" cnn.com
   ```

    Should be 502, as it is not allowed explicitly.

1. Run:
   ```bash
   kubectl exec -n=test-app -ti $APP_B_POD_NAME -- curl -Ls -o /dev/null -w "%{http_code}" google.com
   ```
   
    Should be 200, as we specifically whitelisted `app-b` --> `http(s)://google.com`.

1. Run:
   ```bash
   kubectl exec -n=test-app -ti $APP_B_POD_NAME -- curl -Ls -o /dev/null -w "%{http_code}" cnn.com
   ```

    Should be 502, as it is not allowed explicitly.

1. Run:
   ```bash
   backyards dashboard
   ```

   If you create such traffic as mentioned above and checkout the `test-app` and `external` namespaces on the `TOPOLOGY` view, you should see something like this:
   
   ![](/pics/by-topology.png)
   
   We can see basically the same on the Backyards UI, what we have described as the demo architecture earlier.
   
## Cleanup

1. Delete the demo resources:

   ```bash
   kubectl delete --recursive -f https://github.com/banzaicloud/istio-external-demo/tree/master/resources/
   kubectl delete ns test-app
   kubectl delete ns external
   ```

1. Remove Backyards and Istio from your cluster:

   ```
   backyards uninstall -a
   ```

> If you want to know more about Backyards, check out the [docs](https://banzaicloud.com/docs/backyards), our [blogs](https://banzaicloud.com/tags/backyards/), join the [backyards channel](https://community-banzaicloud.slack.com/archives/CNY0JRMEC) on our Slack or visit the [Backyards product page](https://banzaicloud.com/products/backyards/)!

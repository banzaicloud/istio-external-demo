# Controlling egress traffic with Istio

> This example setup was primarily prepared for this [blog post](https://banzaicloud.com/blog/istio-external-demo/), where you can read more.

We present a working example of how to **control egress traffic** from specific source workloads to specific external services using Istio.

## Use case

Let's take an example use case, where we have two applications in the `test-app` namespace: `app-a` and `app-b`, and we want to achieve the following goals:

1. Deny access towards all external services from the `test-app` namespace by default.
1. Explicitly allow the following application &rightarrow; external service connections:

   - `app-a` &rightarrow; `http(s)://github.com`
   - `app-a` &rightarrow; `http(s)://httpbin.org`
   - `app-b` &rightarrow; `http(s)://google.com`

1. Ensure that all calls to external services go through dedicated egress gateways.
1. Support both HTTPS and plain HTTP calls.

![](/pics/istio-external-demo.png)

To replicate this setup, all you need to do is configure a few Istio resources properly, like egress gateway deployments and services, [Sidecars](https://istio.io/docs/reference/config/networking/sidecar/), [Gateways](https://istio.io/docs/reference/config/networking/gateway/), [VirtualServices](https://istio.io/docs/reference/config/networking/virtual-service/) and [ServiceEntrys](https://istio.io/docs/reference/config/networking/service-entry/), as you can see in this repo.

## Try it out!

### Setup

1. Create a Kubernetes cluster.

   > If you need a hand with that, you can create a cluster with our free version of [Banzai Cloud's Pipeline platform](https://try.banzaicloud.io/).

1. Point `KUBECONFIG` at your cluster.

1. Install Istio with Backyards:

   ```bash
   curl https://getbackyards.sh | sh && backyards install -a
   ```

   In this demo, we install and use [Backyards](https://banzaicloud.com/products/backyards/), the Banzai Cloud Istio distribution, but the resources in this repo can be used with any Istio mesh.
   > Note: [Backyards](https://banzaicloud.com/products/backyards/) is Banzai Cloud's Istio service mesh. You can test and [evaluate](https://banzaicloud.com/products/try-backyards/) it in non-production environments. [Contact us](https://banzaicloud.com/contact/?product_of_interest=Backyards) if you're interested in using Backyards in production.

   This step usually takes a few minutes, and installs Istio and Backyards on your cluster.
1. Create the `test-app` namespace for the example applications:

   ```bash
   kubectl create ns test-app
   ```

1. Enable sidecar-injection for the `test-app` namespace:

   ```bash
   backyards sidecar-proxy auto-inject on test-app
   ```

1. Make sure that the `test-app` namespace has the necessary `istio-injection=enabled` label. (If it is not there yet, wait a few seconds and check again.)

   ```bash
   kubectl get ns test-app --show-labels
   ```

1. Create `external` namespace for the external services. These will be accessible only for specific applications.

   ```bash
   kubectl create ns external
   ```

1. Create all the resources for this demo in the `test-app` and `external` namespaces with the following commands.

   ```bash
   git clone git@github.com:banzaicloud/istio-external-demo.git
   cd istio-external-demo
   kubectl apply --recursive -f resources/
   ```

### Explore

1. Save the application pod names for easier access:

   ```bash
   APP_A_POD_NAME=$(kubectl get pods -n test-app -l k8s-app=app-a -o=jsonpath='{.items[0].metadata.name}')
   APP_B_POD_NAME=$(kubectl get pods -n test-app -l k8s-app=app-b -o=jsonpath='{.items[0].metadata.name}')
   ```

1. Test to make sure that app-a can access httpbin.org:

   ```bash
   kubectl exec -n=test-app -ti $APP_A_POD_NAME -- curl -Ls -o /dev/null -w "%{http_code}" httpbin.org
   ```

    Should be 200, as we specifically whitelisted `app-a` &rightarrow; `http(s)://httpbin.org`.

1. Check that traffic has passed through the egress gateway for the httpbin external service:

   ```bash
   kubectl -n=external logs $(kubectl get pods -n external -l app=egress-httpbin -o=jsonpath='{.items[0].metadata.name}') | tail -n 1
   [2020-04-28T08:07:36.714Z] "GET / HTTP/1.1" 200 - "-" "-" 0 9593 211 211 "10.20.1.179" "curl/7.47.0" "c50400c8-bf41-4c39-9956-ab61deded778" "httpbin.org" "34.230.193.231:80" outbound|80||httpbin.org 10.20.2.4:48588 10.20.2.4:80 10.20.1.179:34648 - -
   ```

1. Test to make sure that app-a can access github.com:

   ```bash
   kubectl exec -n=test-app -ti $APP_A_POD_NAME -- curl -Ls -o /dev/null -w "%{http_code}" github.com
   ```

    Should be 200, as we specifically whitelisted `app-a` &rightarrow; `http(s)://github.com`.

1. Now try to access cnn.com from app-a:

   ```bash
   kubectl exec -n=test-app -ti $APP_A_POD_NAME -- curl -Ls -o /dev/null -w "%{http_code}" cnn.com
   ```

    Should be 502, as it is not allowed explicitly.

1. Test to make sure that app-b can access google.com:

   ```bash
   kubectl exec -n=test-app -ti $APP_B_POD_NAME -- curl -Ls -o /dev/null -w "%{http_code}" google.com
   ```

    Should be 200, as we specifically whitelisted `app-b` &rightarrow; `http(s)://google.com`.

1. Now try to access cnn.com from app-b:

   ```bash
   kubectl exec -n=test-app -ti $APP_B_POD_NAME -- curl -Ls -o /dev/null -w "%{http_code}" cnn.com
   ```

    Should be 502, as it is not allowed explicitly.

1. Check the topology of your mesh on the Backyards dashboard:

   ```bash
   backyards dashboard
   ```

   If you created the traffic we just discussed, check the `test-app` and `external` namespaces in the `TOPOLOGY` view and you should see something like this:

   ![](/pics/istio-external-by-topology.png)

   You can see basically the same thing on the Backyards UI, what we described as "use case architecture" earlier.

   > There is a [feature gap](https://istio.io/news/releases/1.5.x/announcing-1.5/upgrade-notes/#feature-gaps-between-telemetry-v2-and-mixer-telemetry) between the Mixer-based and Mixerless telemetry in Istio, which affects the egress telemetry as well.
   > Backyards comes with [Mixerless Telemetry V2](https://banzaicloud.com/blog/istio-mixerless-telemetry/) by default, so for the sake of this screenshot I switched back to Mixer-based telemetry to gather all necessary egress data.

### Cleanup

1. Delete the demo resources:

   ```bash
   kubectl delete --recursive -f resources/
   kubectl delete ns test-app
   kubectl delete ns external
   ```

1. Remove Backyards and Istio from your cluster:

   ```bash
   backyards uninstall -a
   ```

> If you want to know more about Backyards, check out the [docs](https://banzaicloud.com/docs/backyards/), our [blogs](https://banzaicloud.com/tags/backyards/), join the [backyards channel](https://community-banzaicloud.slack.com/archives/CNY0JRMEC) on our Slack, or visit the [Backyards product page](https://banzaicloud.com/products/backyards/)!

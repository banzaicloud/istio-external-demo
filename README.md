## Demo for restricting access to external services in Istio

1. Create a Kubernetes cluster

1. Point your KUBECONFIG

1. backyards install -a

   free to evaluate, otherwise productpage

1. kubectl create ns test-app

1. backyards sidecar-proxy auto-inject on test-app

1. kubectl create ns external

1. backyards dashboard

1. kubectl apply --recursive -f resources/

1. kubectl exec -n=test-app -ti <APP-A-POD-NAME> -- curl -s -o /dev/null -w "%{http_code}" httpbin.org

1. kubectl exec -n=test-app -ti <APP-A-POD-NAME> -- curl -s -o /dev/null -w "%{http_code}" cnn.com

1. kubectl exec -n=test-app -ti <APP-A-POD-NAME> -- curl -s -o /dev/null -w "%{http_code}" google.com

1. kubectl exec -n=test-app -ti <APP-B-POD-NAME> -- curl -s -o /dev/null -w "%{http_code}" google.com

1. kubectl exec -n=test-app -ti <APP-B-POD-NAME> -- curl -s -o /dev/null -w "%{http_code}" httpbin.org

1. kubectl exec -n=test-app -ti <APP-B-POD-NAME> -- curl -s -o /dev/null -w "%{http_code}" cnn.com

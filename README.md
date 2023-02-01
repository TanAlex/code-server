# README

This repo shows hows to deploy code-server to your GKE  

Reference: 
https://coder.com/docs/code-server/latest/helm

```
git clone https://github.com/coder/code-server
cd code-server
helm upgrade --install code-server ci/helm-chart


NOTES:
1. Get the application URL by running these commands:
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl port-forward --namespace default service/code-server 8080:http

Administrator credentials:

  Password: echo $(kubectl get secret --namespace default code-server -o jsonpath="{.data.password}" | base64 --decode)
```
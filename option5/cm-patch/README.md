# Example for application depedencies in Argo CD

If you use the app-of-apps pattern in Argo CD all applications are deployed in parallel.
Sometimes however you want to deploy them in order

First instruct Argo CD that you want to monitor application health

```
$ kubectl patch cm/argocd-cm --type=merge -n argocd --patch-file patch-argocd-cm.yaml
```

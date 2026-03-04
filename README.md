FluxCD + Helm + k3s

NEED SOPS 

```sh
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=age.agekey
```
# example-kubernetes-goflux-repo

```shell
until kubectl apply -k https://github.com/christianh814/example-kubernetes-go-repo/cluster-XXXX/bootstrap/overlays/default; do sleep 3; done
```
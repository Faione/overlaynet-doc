# helm安装linkerd

[tutorial](https://linkerd.io/2.12/tasks/install-helm/)

add repo

```shell
# To add the repo for Linkerd stable releases:
helm repo add linkerd https://helm.linkerd.io/stable

# To add the repo for Linkerd edge releases:
helm repo add linkerd-edge https://helm.linkerd.io/edge
```

install linkerd crds

```shell
helm install linkerd-crds linkerd/linkerd-crds \
  -n linkerd --create-namespace 
```

install linkerd-control-plane

```shell
helm install linkerd-control-plane \
  -n linkerd \
  --set-file identityTrustAnchorsPEM=ca.crt \
  --set-file identity.issuer.tls.crtPEM=issuer.crt \
  --set-file identity.issuer.tls.keyPEM=issuer.key \
  linkerd/linkerd-control-plane
```
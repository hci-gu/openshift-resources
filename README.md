# openshift-resources

## OC commands and GU endpoints


Command line login
```
oc login --username=gu-x-account --server=https://api.k8s.gu.se:6443
```

Docker login
```
docker login -u unused -p $(oc whoami -t) https://registry.k8s.gu.se
```

## OCI RootSync
```
tar -cf csi.tar helm-charts kustomization.yaml

crane append -f csi.tar -t ${AR_REGION}-docker.pkg.dev/${PROJECT_ID}/csi-kusto/csi-driver:v1

kubectl apply -f rootsync-oci.yaml
```

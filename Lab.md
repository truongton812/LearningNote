1. Lab metalLB

2. Lab helm install prometheus + persistent storage

3. Lab Python custom operator K8s

4. Lab terraform build EC2 + RDS + infra, connect them together

5. Lab CICD AWS

6. Lab DNS + ELB + static page

7. Tạo metric giám sát ổ /data của EC2

8. kustomization + argocd
#nuri-gitops-application/kustomize/overlay/shared-cp-dev2/scp-hyperplane/application/dev2-kr-west1/lcm-common-lma-dev2-kr-west1-hyperplane.yml
```
kind: application
source:
  repoURL:
  path: gitops/lcm-common/kustomize/overlays/lma-dev2-kr-west1-hyperplane
```

#gitops/lcm-common/kustomize/overlays/lma-dev2-kr-west1-hyperplane/kustomization.yml
```
resources:
- ../../base
- resource

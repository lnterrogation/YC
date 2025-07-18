# Развернём кластер Kubernetes на Bridge CNI с помощью YC CLI

```
interrogation@Home ~ % yc managed-kubernetes cluster create \
--name test-k8s \
--network-id <id> \
--public-ip \
--release-channel regular \
--version 1.32 \
--service-account-id <id> \
--master-location zone=ru-central1-a,subnet-id=<id> \
--node-service-account-id <id>
```

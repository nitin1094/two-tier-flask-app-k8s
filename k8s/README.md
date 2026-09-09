# Deploying the two-tier app on a Kubernetes (kubeadm) cluster

These manifests deploy the Flask + MySQL app on a self-managed Kubernetes cluster
(e.g. one built with `kubeadm`). MySQL is backed by a hostPath PersistentVolume;
the Flask app reaches it through the `mysql` Service by name.

## Prerequisites
- A running Kubernetes cluster with `kubectl` configured. Need to build one? Follow
  **Part 2 — Set up a self-managed cluster with kubeadm** in the [root README](../README.md).
- Your app image pushed to a registry, and the `image:` field in
  `two-tier-app-deployment.yml` pointed at it.

## Apply, in order (storage → database → app)
```bash
# 1) Persistent storage for MySQL
kubectl apply -f mysql-pv.yml
kubectl apply -f mysql-pvc.yml

# 2) The database tier
kubectl apply -f mysql-deployment.yml
kubectl apply -f mysql-svc.yml

# 3) The application tier
kubectl apply -f two-tier-app-deployment.yml
kubectl apply -f two-tier-app-svc.yml
```

## Verify
```bash
kubectl get pods
kubectl get svc
```
The `two-tier-app` Service is a NodePort — open `http://<worker-node-ip>:<nodePort>/`.

> **Note:** the manifests use demo database credentials for convenience. Replace them
> with your own before any real use, and prefer a Kubernetes `Secret` (see the
> `eks-manifests/` variant) or an external secrets manager over plaintext values.

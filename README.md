# MongoDB + Mongo Express on Kubernetes

Deploy a MongoDB database with persistent storage and a Mongo Express web UI on Kubernetes. This repo contains all manifests (Deployments, Services, ConfigMaps, Secrets, StorageClass, PVC) needed to run locally or on cloud clusters (AWS/EKS ready).

## Project structure

```text
.
├─ Express/
│  ├─ config-express          # ConfigMap for mongo-express (basic auth + Mongo host)
│  ├─ deploy-express.yml      # Deployment for mongo-express (image: mongo-express)
│  ├─ secret-express          # Secret for mongo-express (Mongo admin creds)
│  └─ svc-express.yml         # LoadBalancer Service for mongo-express (port 8081)
├─ mongodb/
│  ├─ deploy-mongo            # Deployment for MongoDB (image: mongo)
│  ├─ mongo-svc               # ClusterIP Service for MongoDB (port 27017)
│  ├─ pvc-mongo.yml           # PersistentVolumeClaim for MongoDB data
│  ├─ sc-mongo.yml            # StorageClass (AWS EBS CSI)
│  └─ secret-mongo            # Secret for MongoDB root credentials
└─ README.md
```

## Technologies used
- Kubernetes (Deployments, Services, ConfigMaps, Secrets, StorageClass, PVC)
- MongoDB official container image (`mongo`)
- Mongo Express official container image (`mongo-express`)
- AWS EBS CSI driver (provisioner `ebs.csi.aws.com`) for dynamic volumes on EKS

## Prerequisites
- A working Kubernetes cluster and `kubectl` context set
- For AWS/EKS: EBS CSI driver installed and IAM permissions configured
  - Docs: `https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html`
- For other environments (minikube/kind/AKS/GKE): update `mongodb/sc-mongo.yml` to a suitable provisioner or use your cluster's default StorageClass

## Configuration overview
- MongoDB root credentials (base64-encoded in `mongodb/secret-mongo`):
  - `MONGO_INITDB_ROOT_USERNAME`: `mongoadmin`
  - `MONGO_INITDB_ROOT_PASSWORD`: `password123`
- Mongo Express settings:
  - ConfigMap (`Express/config-express`):
    - `ME_CONFIG_MONGODB_SERVER`: `mongo-service`
    - `ME_CONFIG_MONGODB_PORT`: `27017`
    - `ME_CONFIG_BASICAUTH_USERNAME`: `admin`
    - `ME_CONFIG_BASICAUTH_PASSWORD`: `admin123`
  - Secret (`Express/secret-express`):
    - `ME_CONFIG_MONGODB_ADMINUSERNAME` (base64 of `mongoadmin`)
    - `ME_CONFIG_MONGODB_ADMINPASSWORD` (base64 of `password123`)

Note: Update these defaults for any non-demo environment. Encode with:

```bash
echo -n 'your-value' | base64
```

## Deployment (apply in this order)
If desired, create a namespace (optional):

```bash
kubectl create namespace mongo-demo
kubectl config set-context --current --namespace=mongo-demo
```

Apply MongoDB storage and core resources:

```bash
kubectl apply -f mongodb/sc-mongo.yml
kubectl apply -f mongodb/pvc-mongo.yml
kubectl apply -f mongodb/secret-mongo
kubectl apply -f mongodb/deploy-mongo
kubectl apply -f mongodb/mongo-svc
```

Apply Mongo Express resources:

```bash
kubectl apply -f Express/config-express
kubectl apply -f Express/secret-express
kubectl apply -f Express/deploy-express.yml
kubectl apply -f Express/svc-express.yml
```

Verify resources:

```bash
kubectl get pods
kubectl get svc
kubectl get pvc
kubectl get sc
```

## Accessing Mongo Express
- On cloud with `LoadBalancer` (EKS, etc.):
  - Get the external IP/hostname:
    ```bash
    kubectl get svc express-svc -o wide
    ```
  - Open: `http://<EXTERNAL-IP>:8081` (default Service port). If your cloud provider maps NodePort, 8081 will forward to node port `30009`.
- On local clusters (minikube/kind):
  - Port-forward:
    ```bash
    kubectl port-forward svc/express-svc 8081:8081
    ```
    Then open `http://localhost:8081`.

MongoDB is cluster-internal at `mongo-service:27017`.

## Security notes
- Secrets in this repo are sample/demo values. Do not use in production.
- Prefer external secret managers or sealed-secrets. At minimum, rotate credentials and restrict access to the namespace.
- Consider using NetworkPolicies to limit access to MongoDB from only the Express pod or trusted apps.

## Known caveats and tips
- MongoDB volume mount path in `mongodb/deploy-mongo` is set to `/data/mongo`, while the official Mongo image uses `/data/db` by default. If you want true persistence, either:
  - Change `mountPath: /data/mongo` to `mountPath: /data/db`, or
  - Start MongoDB with an explicit `--dbpath /data/mongo` command/args.
- The Express Service is of type `LoadBalancer` and also specifies a `nodePort: 30009`. This is acceptable on many clouds; adjust or remove the `nodePort` if your environment disallows it or if the port is taken.
- The StorageClass uses the AWS EBS CSI provisioner. If you are not on EKS, replace it with your cluster's storage class, or omit and use your default.

## Customization
- Change credentials in `mongodb/secret-mongo` and `Express/secret-express` (remember to base64-encode values).
- Adjust resource limits/requests, liveness/readiness probes, and replica counts for production needs.
- Consider creating an `Ingress` for cleaner external access instead of a `LoadBalancer` Service.
- Use a dedicated namespace (shown above) and RBAC as appropriate.

## Troubleshooting
- Pod stuck in `Pending` with PVC bound issues: ensure a compatible StorageClass exists and your CSI driver is installed.
- Mongo Express shows auth/connection errors: verify that `ME_CONFIG_MONGODB_*` values match `mongo-secret` and that the Service name `mongo-service` is resolvable.
- No external IP for `express-svc`: your cluster may not support `LoadBalancer`. Use port-forwarding or configure an Ingress/LoadBalancer controller.

## Cleanup
Delete resources (reverse order is safest):

```bash
kubectl delete -f Express/svc-express.yml || true
kubectl delete -f Express/deploy-express.yml || true
kubectl delete -f Express/secret-express || true
kubectl delete -f Express/config-express || true

kubectl delete -f mongodb/mongo-svc || true
kubectl delete -f mongodb/deploy-mongo || true
kubectl delete -f mongodb/secret-mongo || true
kubectl delete -f mongodb/pvc-mongo.yml || true
kubectl delete -f mongodb/sc-mongo.yml || true
```

If you created a dedicated namespace:

```bash
kubectl delete namespace mongo-demo
```

**Kubernetes WordPress Project on AWS**

This project deploys a WordPress website with a MySQL database on a multi-node Kubernetes cluster running on AWS EC2.

The project demonstrates Kubernetes Deployments, StatefulSets, Services, persistent storage, ResourceQuota, rolling updates, taints and tolerations, pod affinity, pod anti-affinity, NetworkPolicy, and etcd backup.

**Project Overview**

- Kubernetes cluster created using `kubeadm`
- One control-plane node and two worker nodes
- CRI-O container runtime
- Calico container networking
- WordPress Deployment with 3 replicas
- MySQL StatefulSet with persistent storage
- WordPress exposed through NodePort `30080`
- Namespace: `wp-production`
- StorageClass: `local-rancher-storage`

**Architecture**

```text
Users
  |
Internet
  |
NodePort Service (wordpress-service:30080)
  |
WordPress Deployment (3 Pods)
  |
NetworkPolicy
  |
MySQL Service (mysql-service:3306)
  |
MySQL StatefulSet Pod
  |
PersistentVolumeClaim
  |
StorageClass (local-rancher-storage)
  |
Local Storage Provisioner
```

**Kubernetes Resources**

|           File              |              Purpose                       |
|-----------------------------|--------------------------------------------|
| `namespace.yaml`            | Creates the `wp-production` namespace      |
| `resourcequota.yaml`        | Limits resource usage in the namespace     |
| `storageclass.yaml`         | Creates the local storage class            |
| `pvc.yaml`                  | Requests persistent storage for MySQL      |
| `secret.yaml`               | Provides database configuration values     |
| `mysql-statefulset.yaml`    | Deploys the MySQL database                 |
| `mysql-service.yaml`        | Provides internal access to MySQL          |
| `wordpress-deployment.yaml` | Deploys three WordPress replicas           |
| `wordpress-service.yaml`    | Exposes WordPress using NodePort           |
| `networkpolicy.yaml`        | Allows only WordPress pods to access MySQL |

**Deployment Steps**

**1. Apply the namespace**

```bash
kubectl apply -f namespace.yaml
```

**2. Apply quota and storage resources**

```bash
kubectl apply -f resourcequota.yaml
kubectl apply -f storageclass.yaml
kubectl apply -f pvc.yaml
```

**3. Apply the Secret**

```bash
kubectl apply -f secret.yaml
```

> Do not commit real database passwords to a public GitHub repository. Use a sample Secret or create the Secret separately.

**4. Deploy MySQL**

```bash
kubectl apply -f mysql-service.yaml
kubectl apply -f mysql-statefulset.yaml
```

**5. Deploy WordPress**

```bash
kubectl apply -f wordpress-deployment.yaml
kubectl apply -f wordpress-service.yaml
```

**6. Apply Networkpolicy**

```bash
kubectl apply -f networkpolicy.yaml
```

**Access WordPress**

WordPress is exposed using NodePort `30080`.

```
http://<WORKER-PUBLIC-IP>:30080
```

For security, allow port `30080` in the AWS worker security group only from a trusted IP address.

**Rolling Update**

The WordPress Deployment uses the `RollingUpdate` strategy:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

Update WordPress from version 6.4 to 6.5:

```bash
kubectl set image deployment/wordpress-deployment \
  wordpress=wordpress:6.5 \
  -n wp-production

kubectl rollout status deployment/wordpress-deployment \
  -n wp-production
```

Rollback if required:

```bash
kubectl rollout undo deployment/wordpress-deployment \
  -n wp-production
```

**Taints and Tolerations**

The following taint was applied to `worker02`:

```bash
kubectl taint node worker02 wpnode=true:NoSchedule
```

The WordPress Deployment contains the matching toleration:

```yaml
tolerations:
  - key: wpnode
    operator: Equal
    value: "true"
    effect: NoSchedule
```

This permits WordPress pods to run on the tainted worker node.

**WordPress Pod Affinity**

Preferred pod affinity encourages WordPress pods to run on a node where another WordPress pod is already running.

```yaml
affinity:
  podAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - wordpress
          topologyKey: kubernetes.io/hostname
```

**MySQL Pod Anti-Affinity**

Preferred pod anti-affinity encourages multiple MySQL pods to run on different nodes.

yaml

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - mysql
          topologyKey: kubernetes.io/hostname


The current StatefulSet uses one MySQL replica. This rule becomes visibly effective if multiple database replicas are used.

**NetworkPolicy**

The policy `wordpress-network-policy` selects MySQL pods and allows inbound TCP port `3306` only from pods with the label `app=wordpress` in the `wp-production` namespace.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: wordpress-network-policy
  namespace: wp-production
spec:
  podSelector:
    matchLabels:
      app: mysql
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: wordpress
      ports:
        - protocol: TCP
          port: 3306
```

**ETCD Backup**

The Kubernetes etcd database is backed up on the control-plane node at:

```text
/opt/backup/etcd-backup.db
```

Verify the backup:

```bash
sudo ls -lh /opt/backup/etcd-backup.db
```

The etcd snapshot must not be uploaded to GitHub because it contains the complete cluster state and Kubernetes Secrets.

**Custom WordPress Image**

A custom WordPress image can be built with Podman because the cluster uses CRI-O.

Example `Containerfile`:

```dockerfile
FROM docker.io/library/wordpress:6.5

COPY wp-content/ /var/www/html/wp-content/

RUN chown -R www-data:www-data /var/www/html/wp-content
```

Build the image:

```bash
sudo podman build -t k8s-wordpress:v1 -f Containerfile .
```

The custom image includes copied themes, plugins, and uploads. WordPress users, posts, pages, settings, and passwords remain in the MySQL database and require a separate database backup.

**Verification Commands**

```bash
kubectl get nodes -o wide
kubectl get pods -n wp-production -o wide
kubectl get svc -n wp-production
kubectl get pvc -n wp-production
kubectl get networkpolicy -n wp-production
kubectl describe deployment wordpress-deployment -n wp-production
kubectl describe statefulset mysql-statefulset -n wp-production
```

Test the website:

```bash
curl -I http://<WORKER-IP>:30080
```

An HTTP response such as `200`, `301`, or `302` indicates that WordPress is responding.

**Final Result**

- WordPress runs with three replicas.
- MySQL runs as a StatefulSet with persistent storage.
- WordPress is accessible through a NodePort Service.
- Calico provides pod networking and enforces NetworkPolicy.
- Taints, tolerations, affinity, and anti-affinity control pod scheduling.
- Rolling update and rollback are supported.
- The etcd cluster database is backed up.
- A reusable custom WordPress image can be built with Podman.


**Name**
**Nitin Jangir**

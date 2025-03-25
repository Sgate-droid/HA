# HA

### **What You Need to Uninstall First**
Before reinstalling, **clean up your current setup** on all nodes (master and workers):

1. **Reset Kubernetes** (run on all nodes):
   ```bash
   sudo kubeadm reset --force
   sudo apt remove --purge kubeadm kubectl kubelet kubernetes-cni -y
   sudo rm -rf ~/.kube /etc/kubernetes /var/lib/etcd
   ```

2. **Remove Docker/Containerd** (if no longer needed):
   ```bash
   sudo apt remove --purge docker-ce containerd.io -y
   ```

3. **Clear firewall rules**:
   ```bash
   sudo ufw reset
   sudo iptables -F
   ```

---

### **Correct HA Setup: Step-by-Step Installation**

#### **1. Prerequisites**
- **3 Ubuntu servers** (1 master, 2 workers) with:
  - Static IPs (e.g., `192.168.5.20`, `192.168.5.21`, `192.168.5.22`).
  - SSH access, `sudo` privileges.
  - Minimum specs: 2 CPU cores, 4GB RAM, 20GB disk per node.

#### **2. Install Dependencies (All Nodes)**
```bash
sudo apt update && sudo apt install -y apt-transport-https curl
```

#### **3. Install Kubernetes (All Nodes)**
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update && sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

#### **4. Initialize HA Control Plane (Master Node)**
```bash
sudo kubeadm init --control-plane-endpoint "192.168.5.20:6443" \
  --apiserver-advertise-address 192.168.5.20 \
  --pod-network-cidr 10.244.0.0/16 \
  --upload-certs
```
- Save the `kubeadm join` command for workers.

#### **5. Set Up kubectl (Master)**
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### **6. Install CNI Plugin (Master)**
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

#### **7. Join Worker Nodes**
Run the saved `kubeadm join` command on **both worker nodes**.

#### **8. Verify Cluster**
```bash
kubectl get nodes  # All nodes should show "Ready"
```

---

### **Deploy PostgreSQL HA with Patroni**
#### **1. Install Patroni Operator**
```bash
kubectl create namespace postgres
kubectl apply -f https://raw.githubusercontent.com/zalando/postgres-operator/master/manifests/postgresql.crd.yaml
kubectl apply -f https://raw.githubusercontent.com/zalando/postgres-operator/master/manifests/operator-service-account-rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/zalando/postgres-operator/master/manifests/postgres-operator.yaml
```

#### **2. Deploy PostgreSQL Cluster**
```yaml
apiVersion: "acid.zalan.do/v1"
kind: postgresql
metadata:
  name: postgres-ha
  namespace: postgres
spec:
  teamId: "postgres"
  numberOfInstances: 3
  volume:
    size: 10Gi
  users:
    admin:  # Superuser
      - superuser
      - createdb
  databases:
    testdb: admin
  postgresql:
    version: "14"
```
Apply with:
```bash
kubectl apply -f postgres-ha.yaml
```

#### **3. Verify PostgreSQL HA**
```bash
kubectl get pods -n postgres -l application=spilo
kubectl exec -it postgres-ha-0 -n postgres -- psql -U admin -c "SELECT * FROM pg_stat_replication;"
```

---

### **Deploy Apache with Load Balancing**
#### **1. Create Deployment**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apache
spec:
  replicas: 3
  selector:
    matchLabels:
      app: apache
  template:
    metadata:
      labels:
        app: apache
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - apache
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: apache
        image: httpd:2.4
        ports:
        - containerPort: 80
```

#### **2. Expose with LoadBalancer**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: apache-service
spec:
  selector:
    app: apache
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```
Apply both with:
```bash
kubectl apply -f apache-deployment.yaml
kubectl apply -f apache-service.yaml
```

---

### **Verify HA Functionality**
1. **Simulate Node Failure**:
   ```bash
   kubectl drain node1 --ignore-daemonsets
   ```
   - Verify Apache and PostgreSQL pods reschedule to other nodes.

2. **Test PostgreSQL Failover**:
   ```bash
   kubectl delete pod postgres-ha-0 -n postgres
   kubectl get pods -n postgres -w  # Watch failover
   ```

---

### **Key Takeaways**
- **Kubernetes HA**: Control plane redundancy with `kubeadm init --control-plane-endpoint`.
- **PostgreSQL HA**: Automatic failover with Patroni.
- **Apache HA**: Load balancing + anti-affinity for node resilience.

Let me know if you need help debugging any step!

# Kong API Gateway Deployment Guide (Free Version)

## Prerequisites

* AWS EKS cluster running
* AWS Load Balancer Controller installed in your cluster
* Helm 3 installed
* `kubectl` configured for your cluster

---

## Step-by-Step Deployment

### Step 1: Create Namespace

```bash
kubectl create namespace kong
```

### Step 2: Deploy PostgreSQL Database

```bash
kubectl apply -f postgres-db-setup-fixed.yml
```

### Step 3: Wait for PostgreSQL to be Ready

```bash
kubectl wait --for=condition=ready pod -l app=postgres -n kong --timeout=300s
```

### Step 4: Add Kong Helm Repository

```bash
helm repo add kong https://charts.konghq.com
helm repo update
```

### Step 5: Deploy Kong Control Plane

```bash
helm install kong-cp kong/kong -n kong -f values-cp-fixed.yaml
```

### Step 6: Wait for Kong Deployment

```bash
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=kong -n kong --timeout=300s
```

### Step 7: Create Manager UI Ingress

```bash
kubectl apply -f kong-manager-ingress.yaml
```

### Step 8: Get the External Load Balancer URL

Wait 3–5 minutes for the ALB to be provisioned.

```bash
kubectl get ingress kong-manager-ingress -n kong
kubectl get ingress kong-manager-ingress -n kong -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

---

## Accessing Kong Manager UI

After the ALB is provisioned, get the ALB DNS name:

```bash
echo "Kong Manager URL: http://$(kubectl get ingress kong-manager-ingress -n kong -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
```

Open the URL in your browser.

**Login credentials:**

* **Username:** `kong_admin`
* **Password:** `kong_admin`

---

## Verification Commands

### Check All Pods

```bash
kubectl get pods -n kong
```

### Check Services

```bash
kubectl get svc -n kong
```

### Check Ingress Status

```bash
kubectl get ingress -n kong
```

### Check Kong Admin API (inside cluster)

```bash
kubectl exec -it deployment/kong-cp-kong -n kong -- curl -i http://localhost:8001/
```

### Check Database Connection

```bash
kubectl exec -it deployment/postgres -n kong -- psql -U ps_user -d ps_db -c "\dt"
```

---

## Troubleshooting

### If Pods Are Not Starting

Check pod logs:

```bash
kubectl logs -f deployment/kong-cp-kong -n kong
```

Check events:

```bash
kubectl get events -n kong --sort-by=.metadata.creationTimestamp
```

### If ALB Is Not Provisioned

Check AWS Load Balancer Controller logs:

```bash
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

Verify ingress annotations:

```bash
kubectl describe ingress kong-manager-ingress -n kong
```

---

## Important Notes

* **Database Connection:** Kong connects to PostgreSQL using internal service DNS.
* **No Enterprise Features:** This deployment uses the free open-source version only.
* **Single Instance:** No clustering – best suited for development/testing.
* **Public Access:** Manager UI is accessible from the internet via ALB.
* **No Custom Domain:** Uses ALB’s auto-generated DNS name.


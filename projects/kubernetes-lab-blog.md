# 🚀 Building a Kubernetes Lab: From Pods to Ingress with GitHub Sync

As part of my hands-on Kubernetes lab, I started with a foundational but powerful workflow: setting up a namespace, deploying a pod with both a main and sidecar container, exposing it via a Kubernetes service, and eventually routing external traffic through an NGINX Ingress Controller.

In this post, I’ll walk through the essential steps, explain my choices, and highlight key commands for anyone looking to get started with similar setups.

---

## 🧱 Step 1: Creating a Namespace

Namespaces in Kubernetes help isolate resources. For my lab, I created a dedicated namespace to keep things organized:

```bash
kubectl create namespace lab-app
```

Now, all my resources can live within `lab-app`, ensuring they don’t conflict with anything else in the cluster.

---

## 📦 Step 2: Creating a Pod with a Sidecar Container

In the pod, I wanted two containers:

- **Main Container:** Runs the core application (e.g., a simple HTTP server or a content viewer).
- **Sidecar Container:** Periodically syncs content from a GitHub repository using `git`.

Here's a sample YAML for the pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: content-sync-pod
  namespace: lab-app
spec:
  containers:
    - name: main-app
      image: nginx
      volumeMounts:
        - name: content
          mountPath: /usr/share/nginx/html
    - name: git-sync
      image: k8s.gcr.io/git-sync/git-sync:v3.4.0
      env:
        - name: GIT_SYNC_REPO
          value: "https://github.com/your-user/your-repo.git"
        - name: GIT_SYNC_BRANCH
          value: "main"
        - name: GIT_SYNC_ROOT
          value: "/git"
        - name: GIT_SYNC_DEST
          value: "html"
        - name: GIT_SYNC_ONE_TIME
          value: "false"
        - name: GIT_SYNC_WAIT
          value: "30"
      volumeMounts:
        - name: content
          mountPath: /git
  volumes:
    - name: content
      emptyDir: {}
```

This pattern allows the `main-app` container to serve up-to-date content from GitHub, while the `git-sync` sidecar keeps it fresh.

---

## 🌐 Step 3: Exposing the Pod via a Service

To make the pod reachable within the cluster, I created a ClusterIP service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: content-service
  namespace: lab-app
spec:
  selector:
    app: content-sync-pod
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

> **Note**: Make sure your pod has the correct label (`app: content-sync-pod`) for the service to target it.

---

## 🌍 Step 4: Setting Up Ingress with NGINX Controller

To expose my service outside the cluster (on the host machine), I used an **NGINX Ingress Controller**.

### Step 4.1: Deploy the NGINX Ingress Controller

I installed it using the official Helm chart (Helm must be installed):

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

This sets up a LoadBalancer/NodePort service and the necessary controller pods.

---

### Step 4.2: Create an Ingress Resource

Now I defined an Ingress resource to route traffic based on hostname or path:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: content-ingress
  namespace: lab-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: lab.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: content-service
                port:
                  number: 80
```

> Be sure to add `lab.local` to your `/etc/hosts` file and point it to your host IP or `127.0.0.1` for local testing.

---

### Step 4.3: Handling HTTP and HTTPS Traffic

You can configure HTTPS using a TLS section in the Ingress and a `Secret` containing a certificate and key. To make it dynamic, I used a **ConfigMap** for custom NGINX behaviors:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-custom-config
  namespace: ingress-nginx
data:
  server-snippet: |
    server_tokens off;
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
```

You can link this ConfigMap to the NGINX controller by passing it during Helm install/upgrade:

```bash
helm upgrade nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.configMap.name=nginx-custom-config
```

---

## ✅ Final Test

Once everything is deployed, you can access your content using the hostname (e.g., `http://lab.local`) in your browser. You’ll see content synced from your GitHub repo served by the main container, with traffic routed via the Ingress controller.

---

## 🧪 What’s Next?

In the next phase of my lab, I’ll explore:

- Securing Ingress with HTTPS and Let’s Encrypt via cert-manager  
- Rolling updates for pods  
- Multi-service routing  
- RBAC and network policies  

If you're just getting started with Kubernetes, this setup gives you practical exposure to multiple core concepts — pods, sidecars, services, ingress, Helm, and GitHub integration.

Feel free to try it out and make it your own. Happy learning! 🙌

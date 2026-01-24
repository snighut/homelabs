---

# Pod Port Verification & Connectivity Guide

This guide outlines the process for identifying container ports within the `production` namespace and verifying application connectivity.

## 1. Identify the Container Name

Kubernetes pods can have multiple containers. First, confirm the actual name of the container inside the pod:

```bash
kubectl get pod <POD_NAME> -n production -o jsonpath='{.spec.containers[*].name}'

```

*In our case, the container was named:* `nextjs`.

---

## 2. Locate the Container Port

Once the container name is known, extract the defined `containerPort`:

```bash
kubectl get pod <POD_NAME> -n production -o jsonpath='{.spec.containers[?(@.name=="nextjs")].ports}'

```

**Expected Output:**
`[{"containerPort":3000,"protocol":"TCP"}]`

---

## 3. Verify Local Connectivity (Port Forwarding)

To test if the application is responding correctly without going through a LoadBalancer or Ingress, use a local port forward:

```bash
kubectl port-forward pod/<POD_NAME> -n production 8080:3000

```

| Mapping | Port |
| --- | --- |
| **Local Machine (Mac)** | `8080` |
| **Pod Container** | `3000` |

**Action:** Open your browser and navigate to `http://localhost:8080`.

---

## 4. Production Service Validation

Ensure the Kubernetes Service is correctly routing traffic to the discovered port.

```bash
kubectl get svc -n production

```

Verify that the `TARGETPORT` matches the `containerPort` (3000). If the Service is of type `LoadBalancer` or `NodePort`, it should look like this:

| Service Port | Target Port | Description |
| --- | --- | --- |
| `80` | `3000` | External traffic hits 80, routes to container 3000. |

---

## Useful Debugging Commands

* **Check Logs:** `kubectl logs <POD_NAME> -n production`
* **Check Previous Crash Logs:** `kubectl logs <POD_NAME> -n production -p`
* **Describe Pod Status:** `kubectl describe pod <POD_NAME> -n production`

---

Would you like me to help you generate a `service.yaml` file based on these port settings?
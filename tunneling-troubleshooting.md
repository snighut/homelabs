---

# 🛠 Cloudflare Tunnel Troubleshooting Guide

## 1. How the Tunnel Works

A Cloudflare Tunnel (powered by `cloudflared`) creates a **secure outbound-only connection** from your home server to the Cloudflare Global Network.

* **No Open Ports:** You do not need to open port 80 or 443 on your home router.
* **The Handshake:** When the pod starts, it uses the `TUNNEL_TOKEN` to authenticate. It then establishes four connections (for high availability) to the nearest Cloudflare edge data centers.
* **Traffic Flow:** User -> `https://your-domain.com` -> Cloudflare Edge -> **Established Tunnel** -> `cloudflared` Pod -> Kubernetes Service -> Your Next.js App.

---

## 2. Verify & Validate the Token

If the tunnel won't connect, you must verify the token stored in your cluster.

### Retrieve and Decode

Standard Cloudflare tokens are **Base64-encoded JSON**. You can decode them to check if the **Tunnel ID** matches your dashboard.

```bash
# Get the secret and decode it
kubectl get secret cloudflared-token -o jsonpath="{.data.token}" | base64 --decode

```

### Validation Checklist

* **The JSON structure:** You should see `{"a":"account_id","t":"tunnel_id","s":"secret"}`.
* **Match the ID:** The value of `"t"` must match the **Tunnel ID** shown in the Cloudflare Dashboard under **Networks > Tunnels**.
* **No Hidden Characters:** If the command above outputs a `%` or extra spaces, your token is likely corrupted with shell-formatting characters.

---

## 3. Verify via Logs

The logs are the ultimate source of truth. Use this command on your local machine to see the connection status:

```bash
kubectl logs -l app=cloudflared -f

```

### Log Status Meanings

| Log Message | Status | Meaning |
| --- | --- | --- |
| `Registered tunnel connection` | **SUCCESS** ✅ | The tunnel is live and ready for traffic. |
| `Unauthorized: Failed to get tunnel` | **AUTH ERROR** ❌ | Token is wrong, or the node's clock is out of sync. |
| `Retrying connection...` | **NET ERROR** ❌ | DNS or firewall is blocking outbound TCP/UDP. |

---

## 4. Resetting from Scratch (Bulletproof Method)

If you are stuck in an "Unauthorized" loop, follow this **"Sanitized File"** method to ensure no hidden characters enter your cluster.

1. **Delete the Tunnel in the Dashboard:** Create a brand new one to get a fresh secret.
2. **Create a Sanitized Token File (on your local machine):**
```bash
# Paste your new token inside the single quotes
echo -n 'YOUR_NEW_TOKEN' | tr -dc '[:alnum:]._-' > clean.token

```


3. **Replace the Secret:**
```bash
kubectl delete secret cloudflared-token --ignore-not-found
kubectl create secret generic cloudflared-token --from-file=token=clean.token
rm clean.token

```


4. **Restart the Pod:**
```bash
kubectl rollout restart deployment cloudflared-bridge

```



---

## 5. Additional Troubleshooting Solutions

| Problem | Solution |
| --- | --- |
| **Clock Skew** | Ensure Talos is synced. Run `talosctl get timestatus`. If `Synced` is false, Cloudflare will reject the token as "expired." |
| **Protocol Issues** | Force TCP by adding `--protocol http2` to the `cloudflared` arguments. Many ISP routers block the default UDP (QUIC) protocol. |
| **SSL Redirect Loop** | If your app only speaks HTTP (standard for K8s pods), set Cloudflare SSL to **"Full"** (not Full Strict). |
| **1033 Error** | Usually means the tunnel is connected, but the **Public Hostname** in the dashboard isn't pointed to the correct internal service. |
| **Pod Security Violation** | In restricted Talos clusters, ensure the pod has a `securityContext` that drops all capabilities and runs as a non-root user (e.g., `65532`). |

---

### Next Step for the User

**Would you like me to help you set up a "Health Check" notification so you get an email if your GMKtec K8 Plus ever goes offline?**
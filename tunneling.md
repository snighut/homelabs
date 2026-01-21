
---

# 🌐 How Our Tunneling Infrastructure Works

This project uses **Cloudflare Tunnel** to expose the `collaboration-app` to the internet. Unlike traditional methods that require opening ports on your home router (Port Forwarding), a tunnel works by establishing a secure, outbound-only connection from within the home lab.

## 1. What is Tunneling?

In networking, **tunneling** is a method where one network protocol (the "guest" protocol) is encapsulated within another (the "delivery" protocol).

In our setup, the **"delivery"** protocol is standard HTTPS (TCP/443). The tunnel creates a virtual bridge that makes your **home server** feel like it's sitting directly inside Cloudflare's global data center, even though it is behind your home router and a firewall.

---

## 2. End-to-End Request Flow

When a user visits `https://your-domain.com`, the request performs a "traversal" across several layers:

### Step A: The Edge (User to Cloudflare)

1. **DNS Lookup:** The user's browser asks for the IP of `your-domain.com`. Cloudflare DNS returns a Cloudflare Anycast IP (not your home IP!).
2. **The Handshake:** The browser establishes a TLS connection with the nearest Cloudflare data center (the "Edge").
3. **WAF & Security:** Cloudflare checks for bots, DDoS attacks, or firewall rules.

### Step B: The Bridge (Cloudflare to Home Server)

4. **Tunnel Lookup:** Cloudflare sees that `your-domain.com` is managed by a tunnel.
5. **Encapsulation:** Cloudflare wraps the user's HTTP request and sends it down the **pre-established outbound connection** maintained by your `cloudflared` pod.
6. **NAT Traversal:** Because the connection was started from *inside* your house, your home router's firewall allows the data to return freely. This "punches a hole" without opening any ports.

### Step C: The Cluster (Home Server to Pod)

7. **Decapsulation:** The `cloudflared` pod on your K8s cluster receives the data and unwraps the original HTTP request.
8. **Internal Routing:** `cloudflared` looks at its "Public Hostname" config and sees the request should go to `collaboration-app-service.default.svc.cluster.local`.
9. **The Target:** The Kubernetes Service routes the traffic to one of the active **Next.js Pods**.
10. **The Response:** Your app sends the HTML back through the same tunnel, back to Cloudflare, and finally to the user's screen.

---

## 3. Why this is "Production Grade"

This architecture, running on your **home server**, offers several advantages:

* **Zero Attack Surface:** Since no ports are open on your router, a hacker scanning your home IP address sees nothing.
* **Identity-Based Access:** We can place a "Cloudflare Access" gate in front of the tunnel, requiring an email login before anyone can even reach the Next.js app.
* **Performance:** Traffic enters the Cloudflare network at the data center closest to the user and stays on their high-speed fiber backbone until it reaches the data center closest to your house.

---

## 4. Key Components in this Repo

| Component | Responsibility |
| --- | --- |
| **`cloudflared` Pod** | Maintains the "heartbeat" with Cloudflare and routes incoming traffic. |
| **`TUNNEL_TOKEN`** | The cryptographically signed key that authorizes the pod to represent your domain. |
| **`ClusterIP` Service** | The internal-only mailbox that your Next.js app uses to receive traffic from the tunnel. |

---

### Maintenance Note

If you ever see a **1033 Error** or **502 Bad Gateway**, it means the "Bridge" is broken. Refer to the `troubleshoot.md` file to check the `cloudflared` logs and verify the **Tunnel Token**.

**Would you like me to help you add a "Monitoring" section to this README that shows how to track how many users are visiting your app through the tunnel?**
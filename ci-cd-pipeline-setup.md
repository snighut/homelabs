This README provides a comprehensive overview of the Automated Image Update pipeline used for the **collaboration-app** on our **GMKtec K8 Plus** production cluster.

---

# CI/CD Pipeline & GitOps Documentation

This document outlines the end-to-end automation flow—from a developer pushing code on an **M4 Mac Mini** to the automatic deployment on our **Production Kubernetes cluster**.

## 1. System Architecture Overview

Our pipeline follows the **GitOps** pattern using GitHub Actions for CI (Continuous Integration) and FluxCD for CD (Continuous Deployment).

---

## 2. CI: GitHub Actions & Docker Build

### Triggering the Build

A build is triggered automatically whenever a developer pushes code to the `main` branch of the `collaboration-app` repository.

### Image Creation & Tagging Strategy

The build is handled by **GitHub Actions** using the `docker/build-push-action`. To allow Flux to identify new versions, we use a dual-tagging strategy:

1. **`latest`**: Always points to the most recent successful build.
2. **`YYYYMMDD-HHMM` (Timestamp)**: A unique, immutable tag (e.g., `20260121-1620`).

**Why unique tags?** Flux requires unique, alphabetically sortable strings to identify that a *new* image exists. Using only `latest` would prevent Flux from detecting a change in the manifest.

---

## 3. CD: FluxCD Automation Loop

Once the image is on Docker Hub, the **FluxCD controllers** on the GMKtec K8 Plus take over.

### Step A: Identification (Image Scanning)

The `ImageRepository` resource scans Docker Hub every few minutes. It maintains a list of all available tags for `sbnighut/collaboration-app`.

### Step B: The Policy Match

The `ImagePolicy` defines which tag should be running in production. It uses a **Regex Filter** to ignore the `latest` tag and focus on timestamped tags:

* **Filter:** `^202[0-9].*` (matches tags starting with the year).
* **Policy:** `alphabetical: order: asc` (the "highest" alphabetical string wins).

### Step C: The Manifest Update (Git Write-Back)

When the `ImagePolicy` identifies a new "winner" (e.g., a newer timestamp), the `ImageUpdateAutomation` controller:

1. Clones the `fleet-infra` repository.
2. Locates the **Setter Marker** in the `app.yaml`:
```yaml
image: sbnighut/collaboration-app:latest # {"$imagepolicy": "flux-system:collaboration-app-policy"}

```


3. Rewrites the file, replacing the old tag with the new one.
4. Commits and pushes the change back to GitHub (requires **Write Access** on the Deploy Key).

---

## 4. Final Deployment: Git to Kubernetes

Flux's `Kustomization` controller watches the `fleet-infra` repository.

1. It detects the new commit created by the Image Automator.
2. It pulls the updated `app.yaml`.
3. It applies the changes to the cluster using `kubectl apply` logic.
4. **Kubernetes** performs a rolling update, pulling the new image tag from Docker Hub and restarting the pods.

---

## 5. Troubleshooting & Commands

If the pipeline is stuck, verify the "Chain of Custody" with these commands:

| Step | Command | Goal |
| --- | --- | --- |
| **Scanner** | `flux get image repository collaboration-app` | Is Flux seeing the new tags on Docker Hub? |
| **Policy** | `flux get image policy collaboration-app-policy` | Is the new tag identified as the "Winner"? |
| **Writer** | `flux get image update collaboration-app-automation` | Was the commit successfully pushed to GitHub? |
| **Deploy** | `kubectl get pods -n production` | Is the new pod actually running? |

> **Note:** Ensure the GitHub Deploy Key has **Allow Write Access** enabled, otherwise the Automator will fail with a `read only` error during the Git Push phase.

---
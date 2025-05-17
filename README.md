
# Admission Controllers for Default Storage Class & Resource Limits in AKS Using Python Flask

This project demonstrates how to implement Kubernetes admission controllers using a Python Flask webhook. The use case covers two scenarios:
- **Default Storage Class:** Automatically assign a default storage class to PersistentVolumeClaims (PVCs) that do not specify one.
- **Default Resource Limits:** Automatically inject default resource limits and requests into Pods that are missing such configurations.

Using these admission controllers in AKS helps enforce consistency and resource governance across your cluster.

---

## Overview

Kubernetes admission controllers allow you to intercept, validate, or modify API requests before they are persisted. In this example, we implement two mutating webhooks:
1. **PVC Mutator:** If a PVC request does not specify a `storageClassName`, inject a patch to set it to `default-storage`.
2. **Pod Mutator:** If a Pod creation request does not include resource limits/requests in one or more containers, inject a patch to add default resource values (e.g., CPU and memory).

The webhook is built using Python Flask and returns JSON patches (base64‑encoded) that are applied by the Kubernetes API server.

---

## Prerequisites

- **AKS Cluster:** A running Azure Kubernetes Service cluster.
- **kubectl:** Configured to access your AKS cluster.
- **Python 3.9+ & pip:** For developing the Flask webhook.
- **Docker:** To containerize the Flask application.
- **TLS Certificates:** For production use, secure your webhook with proper TLS (here we assume `cert.pem` and `key.pem`).

---

## Step 1: Write the Python Flask Webhook Server

Create a file named `webhook.py` with the content below. This script defines two endpoints: one to mutate PVCs and one to mutate Pods.

```python
import base64
import json
from flask import Flask, request, jsonify

app = Flask(__name__)

# Endpoint to enforce a default storage class on PersistentVolumeClaims (PVCs)
@app.route("/mutate-pvc", methods=["POST"])
def mutate_pvc():
    req = request.get_json()
    pvc = req["request"]["object"]
    patch_ops = []

    # If the PVC spec does not contain a storageClassName, add the default.
    if "storageClassName" not in pvc.get("spec", {}):
        patch_ops.append({
            "op": "add",
            "path": "/spec/storageClassName",
            "value": "default-storage"
        })

    if patch_ops:
        patch_str = base64.b64encode(json.dumps(patch_ops).encode()).decode()
        response = {
            "response": {
                "uid": req["request"]["uid"],
                "allowed": True,
                "patchType": "JSONPatch",
                "patch": patch_str
            }
        }
        return jsonify(response)
    else:
        return jsonify({"response": {"uid": req["request"]["uid"], "allowed": True}})

# Endpoint to enforce default resource limits on Pods
@app.route("/mutate-pod", methods=["POST"])
def mutate_pod():
    req = request.get_json()
    pod = req["request"]["object"]
    patch_ops = []
    containers = pod.get("spec", {}).get("containers", [])
    
    # Loop through each container and add resource limits if not provided.
    for i, container in enumerate(containers):
        if "resources" not in container or "limits" not in container["resources"]:
            patch_ops.append({
                "op": "add",
                "path": f"/spec/containers/{i}/resources",
                "value": {
                    "limits": {"cpu": "500m", "memory": "512Mi"},
                    "requests": {"cpu": "250m", "memory": "256Mi"}
                }
            })

    if patch_ops:
        patch_str = base64.b64encode(json.dumps(patch_ops).encode()).decode()
        response = {
            "response": {
                "uid": req["request"]["uid"],
                "allowed": True,
                "patchType": "JSONPatch",
                "patch": patch_str
            }
        }
        return jsonify(response)
    else:
        return jsonify({"response": {"uid": req["request"]["uid"], "allowed": True}})

if __name__ == "__main__":
    # For production use, provide valid TLS certificates here.
    app.run(host="0.0.0.0", port=443, ssl_context=("cert.pem", "key.pem"))
```

*Explanation:*
- The `/mutate-pvc` endpoint checks incoming PVC creation requests. If `storageClassName` is missing, it returns a JSON patch to add it.
- The `/mutate-pod` endpoint loops through Pod containers and injects resource limits/requests when missing.
- The patches are base64-encoded to comply with the AdmissionReview API spec.

---

## Step 2: Create Kubernetes Webhook Configurations

These YAML files configure Kubernetes to forward admission requests to your webhook service.

### A. MutatingWebhookConfiguration for PVCs

Create a file named `mutating_webhook_pvc.yaml`:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: default-storage-class-webhook
webhooks:
  - name: default-storage.k8s.io
    clientConfig:
      service:
        name: storage-webhook-service
        namespace: default
        path: "/mutate-pvc"
      caBundle: <CA_BUNDLE>  # Replace with your base64-encoded CA certificate if needed
    rules:
      - operations: ["CREATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["persistentvolumeclaims"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
```

### B. MutatingWebhookConfiguration for Pods

Create a file named `mutating_webhook_pod.yaml`:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: default-resource-limits-webhook
webhooks:
  - name: default-resources.k8s.io
    clientConfig:
      service:
        name: resource-webhook-service
        namespace: default
        path: "/mutate-pod"
      caBundle: <CA_BUNDLE>  # Replace with your base64-encoded CA certificate if needed
    rules:
      - operations: ["CREATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
```

*Note:* Replace `<CA_BUNDLE>` with your certificate data for secure communication if applicable.

---

## Step 3: Deploy the Flask Webhook Server in AKS

### A. Create a Dockerfile

Create a file named `Dockerfile` with the following content:

```dockerfile
FROM python:3.9

WORKDIR /app
COPY webhook.py .

# Install required Python packages
RUN pip install flask

# Expose port 443 for HTTPS traffic
EXPOSE 443

# Start the Flask webhook server
CMD ["python", "webhook.py"]
```

### B. Build and Push the Docker Image

Replace `myrepo` with your container registry name (e.g., Docker Hub or ACR):

```bash
docker build -t myrepo/admission-webhook:latest .
docker push myrepo/admission-webhook:latest
```

### C. Create a Kubernetes Deployment

Create a file named `webhook_deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admission-webhook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: admission-webhook
  template:
    metadata:
      labels:
        app: admission-webhook
    spec:
      containers:
        - name: admission-webhook
          image: myrepo/admission-webhook:latest
          ports:
            - containerPort: 443
```

Deploy the webhook server:

```bash
kubectl apply -f webhook_deployment.yaml
```

### D. Create a Kubernetes Service

Create a file named `webhook_service.yaml` to expose the webhook server:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: storage-webhook-service
  namespace: default
spec:
  selector:
    app: admission-webhook
  ports:
    - protocol: TCP
      port: 443
      targetPort: 443
---
apiVersion: v1
kind: Service
metadata:
  name: resource-webhook-service
  namespace: default
spec:
  selector:
    app: admission-webhook
  ports:
    - protocol: TCP
      port: 443
      targetPort: 443
```

Apply the service configuration with:

```bash
kubectl apply -f webhook_service.yaml
```

---

## Step 4: Testing the Webhooks

1. **Test the Default Storage Class Webhook:**
   - Create a PersistentVolumeClaim without specifying the `storageClassName`.
   - Run `kubectl get pvc -o yaml` to verify that `storageClassName` has been set to `default-storage`.

2. **Test the Default Resource Limits Webhook:**
   - Create a Pod without resource limits in its container specification.
   - Run `kubectl get pod <pod-name> -o yaml` to ensure that default resource limits (`cpu: 500m, memory: 512Mi` for limits and `cpu: 250m, memory: 256Mi` for requests) have been injected.

---

## Conclusion

By following these steps, you have implemented two mutating admission controllers in AKS using a Python Flask webhook. This setup ensures that:
- All PVCs receive a default storage class when none is specified.
- All Pods are deployed with default resource limits and requests if not explicitly defined.

For further details, consult:
- [Kubernetes Admission Controllers Documentation](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [Flask Documentation](https://flask.palletsprojects.com/)
- [Docker Documentation](https://docs.docker.com/)


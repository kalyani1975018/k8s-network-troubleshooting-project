# Kubernetes Network Troubleshooting: frontend → backend

## Overview

I built a small `client → frontend → backend` setup on Kubernetes and created one networking problem. Then I investigated it like a support ticket, found the root cause and fixed it.

**The problem I created:** a wrong `targetPort` in the backend Service. The backend pod is reachable directly, but not through the Kubernetes Service.

## Tools Required

| Tool | Why I need it |
|------|---------------|
| Docker Desktop | Runs containers. Kind builds the cluster inside Docker. |
| Kind | Creates the local Kubernetes cluster |
| kubectl | Command line tool to work with the cluster |
| Git | Pushes the project to GitHub |
| VS Code | Editor and terminal |

Check that the tools are installed:
```bash
docker run hello-world
kind --version
kubectl version --client
git --version
```

## Environment

| Item | Details |
|------|---------|
| Cluster | Kind cluster named `demo-cluster` |
| Namespace | `test-ns` |
| client | A test pod (`nicolaka/netshoot`) with curl, nslookup and ping |
| frontend | An nginx pod with a ClusterIP Service on port 80 |
| backend | 2 pods running my own Java app on Apache TomEE (port 8080), with a ClusterIP Service |

**Flow**
```
client → Service "frontend" → frontend pod (nginx)
frontend pod → Service "backend" → backend pods (TomEE, port 8080)
```

## Files in this repository

```
k8s-network-troubleshooting/
├── README.md
├── manifests/
│   ├── namespace.yaml
│   ├── backend.yaml
│   ├── frontend.yaml
│   ├── client.yaml
│   └── backend-service-update.yaml
```

| File | Purpose |
|------|---------|
| `namespace.yaml` | Creates the namespace `test-ns` |
| `backend.yaml` | Creates 2 backend pods and the correct backend Service (port 80 → targetPort 8080) |
| `frontend.yaml` | Creates the nginx frontend pod and its Service |
| `client.yaml` | Creates the client test pod. It runs `sleep infinity` so it stays alive. |
| `backend-service-update.yaml` | The backend Service with a wrong port (targetPort 9090). Applying it creates the problem. |

**Service ports, in simple words**
- `port` is the port other pods call (80).
- `targetPort` is the port the application really listens on. For the backend it must be **8080**.

## Setup

**1. Create the cluster**
```bash
kind create cluster --name demo-cluster
kubectl get nodes
```
The node shows `Ready`.

**2. Apply the files**
```bash
kubectl apply -f manifests/namespace.yaml
kubectl apply -f manifests/backend.yaml
kubectl apply -f manifests/frontend.yaml
kubectl apply -f manifests/client.yaml
```

**3. Check that the pods are running**
```bash
kubectl get pods -n test-ns -o wide
```
All pods show `Running`.

## Baseline: everything works

**client → frontend**
```bash
kubectl exec -n test-ns client -- curl -s http://frontend
```
Result: the nginx welcome page.

**frontend → backend** (run from inside the frontend pod)
```bash
kubectl exec -n test-ns deploy/frontend-deployment -- wget -S -O /dev/null http://backend/testapp
```
Result: `HTTP/1.1 200` from `Apache TomEE`. The connection works.

**Endpoints**
```bash
kubectl get endpoints backend -n test-ns
```
Result: the backend pods with port `8080`.

## Create the problem

I apply a second version of the backend Service. It changes only one value: `targetPort` from `8080` to `9090`. The pods are not changed.

```bash
kubectl apply -f manifests/backend-service-update.yaml
```

## Symptom

**What was failing?**

| Path | Result |
|------|--------|
| client → frontend | Works |
| frontend → backend | **Fails with "Connection refused"** |

```bash
kubectl exec -n test-ns client -- curl -s http://frontend
kubectl exec -n test-ns deploy/frontend-deployment -- wget -S -O /dev/null http://backend/testapp
```
The first command works. The second fails:
```
wget: can't connect to remote host: Connection refused
```

"Connection refused" means the request reached the address, but nothing is listening on that port. This points to a wrong port, not to DNS.

## Investigation

**What commands did I use?** I checked one layer at a time.

**1. Are the pods running?**
```bash
kubectl get pods -n test-ns -o wide
```

**2. Does DNS work from the frontend pod?**
```bash
kubectl exec -n test-ns deploy/frontend-deployment -- nslookup backend.test-ns.svc.cluster.local.
```
**3. Is the pod reachable directly?** (copy a backend pod IP from command 1, replace `<POD_IP>`)
```bash
kubectl exec -n test-ns client -- curl -s -o /dev/null -w "%{http_code}\n" http://<POD_IP>:8080/testapp/
```

**4. Is it reachable through the Service?**
```bash
kubectl exec -n test-ns client -- curl -s --max-time 5 -o /dev/null -w "%{http_code}\n" http://backend/testapp/
```

**3. Does the Service exist?**
```bash
kubectl get svc -n test-ns
```

**4. Which port do the endpoints use?**
```bash
kubectl get endpoints backend -n test-ns
```

**5. Which port does the application listen on?**
```bash
kubectl logs -n test-ns deploy/backend-deployment --tail=3
```

**6. Is a NetworkPolicy blocking traffic?**
```bash
kubectl get networkpolicy -n test-ns
```

## Observations

**What did I discover?**

- All pods were `Running` and `Ready`, so this was not a pod failure.
- DNS resolved the `backend` name correctly, so it was not a DNS problem.
- The pod answered with `200` when called directly on port 8080
- Through the Service, the connection was refused.
- The endpoints showed port **9090**.
- The backend logs showed the application listening on port **8080**.
- No NetworkPolicy existed, so nothing was blocking the traffic.

**In short:** the pod is reachable directly, but not through the Kubernetes Service.

## Root Cause

**What exactly caused the failure?**

The backend Service had `targetPort: 9090`, but the application listens on port **8080**.

The Service sent traffic to a port where nothing was listening, so every connection was refused. The pods were healthy the whole time. Only the Service configuration was wrong.

## Resolution

**What did I change?**

I set `targetPort` back to `8080` by applying the correct Service definition:

```bash
kubectl apply -f manifests/backend.yaml
```
Output: `service/backend configured`.

## Verification

**How did I verify the fix?**

**1. The endpoints use the correct port**
```bash
kubectl get endpoints backend -n test-ns
```
They show port `8080`.

**2. frontend → backend works**
```bash
kubectl exec -n test-ns deploy/frontend-deployment -- wget -S -O /dev/null http://backend/testapp
```
Result: `HTTP/1.1 200` from `Apache TomEE`.

**3. client → frontend still works**
```bash
kubectl exec -n test-ns client -- curl -s -o /dev/null -w "%{http_code}\n" http://frontend
```
Result: `200`.

| Check | Before the fix | After the fix |
|-------|----------------|---------------|
| Endpoint port | 9090 | 8080 |
| Service `TargetPort` | 9090 | 8080 |
| frontend → backend | Connection refused | HTTP 200 |

## Prevention

- After any Service change, check that the endpoint port matches the application port.

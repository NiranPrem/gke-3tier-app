Networking

🔹 1. Inside Kubernetes Pod Networking

Every Pod in your cluster gets an internal IP (from the GKE Pod CIDR range).

Pods can talk to each other directly, regardless of which node they’re running on.

Example:

frontend-blue pod → IP 10.x.x.x

backend-green pod → IP 10.x.x.x

They can talk directly because GKE uses a flat network.

🔹 2. Services (Stable Access)

Pods are ephemeral (they can die/restart). To give them a stable address, you expose them using Services.

ClusterIP (default):

Only reachable inside the cluster.

Example: your backend-svc → ClusterIP.

Frontend can reach it using http://backend-svc:5000.

But you cannot reach it from your laptop.

LoadBalancer:

Creates a Google Cloud Load Balancer + external IP.

Example: frontend-svc → External IP 35.193.184.157.

You can open this from your browser → http://35.193.184.157.

NodePort (not used here):

Exposes service on each Node’s IP and a high port (30000-32767).

Mostly for dev/test.

🔹 3. Your Current Setup

From your kubectl get svc output:

backend-svc    ClusterIP      34.118.230.243   <none>           5000/TCP
frontend       LoadBalancer   34.118.225.90    34.71.16.27      80:31127/TCP
frontend-svc   LoadBalancer   34.118.236.154   35.193.184.157   80:30613/TCP


frontend and frontend-svc:

Both are type LoadBalancer.

Both have different external IPs (34.71.16.27 and 35.193.184.157).

You can hit either IP in your browser to reach frontend.

Probably you don’t need both; maybe one is leftover from earlier deploys.

backend-svc:

ClusterIP only.

Not reachable from outside, only from frontend pods.

This is actually good — you normally don’t want the DB/API exposed to the internet.

🔹 4. Blue/Green Traffic Switching

Right now:

You have two deployments per app (frontend-blue, frontend-green, backend-blue, backend-green).

Only one set is active for clients, depending on which label is selected in frontend-svc and backend-svc.

Example:

If frontend-svc → app: frontend-blue → traffic flows only to blue pods.

If you patch → app: frontend-green → traffic switches to green pods instantly.

🔹 5. Traffic Flow (End-to-End)

User opens → http://35.193.184.157 (LoadBalancer IP).

Request goes to Google Cloud Load Balancer.

LB forwards request to your GKE Service (frontend-svc).

Service forwards traffic → matching Pods (frontend-blue or frontend-green).

Frontend Pod → sends API request to backend-svc:5000 (ClusterIP).

Backend Pod answers → DB (if you have one).

Response flows back → user’s browser.

🔹 6. Visualization
Your Browser
     │
     ▼
 [ Google Cloud LoadBalancer ]
     │  (External IP: 35.193.184.157)
     ▼
 frontend-svc (Service: LoadBalancer)
     │
     ├──> frontend-blue Pod(s)
     └──> frontend-green Pod(s)
               │
               ▼
       backend-svc (Service: ClusterIP)
               │
               ├──> backend-blue Pod(s)
               └──> backend-green Pod(s)


👉 So:

Frontend is public (via LoadBalancer).

Backend is private (via ClusterIP).

Blue/Green switching is controlled by which Deployment your Service points to.

# Implementation Explanation

## 1. Choice of Kubernetes Objects for Deployment
- Used Deployments for frontend and backend (stateless, easy scaling). 
- For database, used StatefulSet for stable pod identity (mongo-0, mongo-1) and ordered deployment, ideal for storage apps. 
- Replicas=2 across all for fault tolerance—controllers heal by recreating pods.
-  Labels (e.g., app: mongo) for selectors and tracking.
- Added PodDisruptionBudget (PDB) for mongo: Ensures maxUnavailable: 0 pod during voluntary disruptions (e.g., autoscaling, maintenance). Matches StatefulSet selector (app: mongo). This protects against simultaneous evictions, maintaining availability in autoscaling clusters (min 3 nodes).
- Enhanced mongo StatefulSet with readiness probe (mongosh ping) to ensure pods are truly ready, improving PDB effectiveness during disruptions.

## 2. Method Used to Expose Pods to Internet Traffic
- Frontend: LoadBalancer Service provisions a public IP on port 3000 (targets container 3000).Chosen because GKE provisions an external IP automatically, ideal for internet-facing apps 
- Backend and mongo: ClusterIP for internal traffic only (backend on 5000, mongo on 27017). 
- Frontend proxies API calls to backend..

## 3. Use of Persistent Storage
- Yes, via PVC in StatefulSet (1Gi per pod, GKE 'standard' class). 
- Data at /data/db persists across pod deletions/recreations. 
- Replica set ensures data replication between pods.

## 4. Git Workflow
- New repo created.
- Commits: Incremental (e.g., "Add mongo StatefulSet", 10+ total) with descriptive messages.
- Structure: manifests/ folder for YAMLs.
- Pushed to main after each major change.

## 5. Successful Running and Debugging
- App runs at README URL: **Live Link** http://34.132.220.202:3000/
- UI loads on port 3000, API calls to backend:5000, DB connects via replica set URI. 
- Cart adds/persists. 
- Debugging: Used `kubectl logs/describe` for issues (e.g., init container errors). 
- fixed network errors via relative URLs and Nginx proxy in frontend image v2.1.0
- Self-healing tested by deleting pods—recreated in seconds, no data loss.

## 6. Good Practices
- Image tags: Versioned (e.g., :1.0) for identification.
- Resources: Requests/limits on all containers for efficient scheduling.
- Replicas=2: Ensures availability.
- Modular manifests for maintainability.
- Numbered manifests for ordered apply (DB first).


## 7. How Pods Communicate Through Services
- Pods are ephemeral; services provide stable endpoints. Mongo service (headless) allows direct access to pods (mongo-0.mongo:27017). Backend service routes to backend pods via labels. Frontend pods call backend service name internally. LoadBalancer exposes frontend externally.
- Why LoadBalancer for frontend: Simplest for public access in GKE—auto-provisions IP, handles traffic routing without manual ingress setup.

## 8.0 Summary of Each Manifest File

- **01-mongo-statefulset.yaml:** 
    - Kind: StatefulSet, name: mongo, replicas:2, labels: app:mongo. 
    - InitContainer for replica set setup. 
    - Container: mongo:6.0, port:27017, resources (requests:200m CPU/256Mi mem, limits:500m/512Mi), volumeMount: /data/db. volumeClaimTemplates: 1Gi PVC.
    -Added readinessProbe to mongo container: exec mongosh ping, with delays/thresholds for health checks.
- **02-mongo-service.yaml:**
    - Kind: Service, name: mongo, headless (clusterIP:None), selector: app:mongo, port:27017.
- **03-backend-deployment.yaml:** 
    - Kind: Deployment, name: backend, replicas:2, labels: app:backend. 
    - Container: yourusername/yolo-backend:1.0, port:5000, env: MONGO_URI to replica set, resources (100m/128Mi requests, 300m/256Mi limits).
- **04-backend-service.yaml:** 
    - Kind: Service, name: backend, selector: app:backend, port:5000.
- **05-frontend-deployment.yaml:** 
    - Kind: Deployment, name: frontend, replicas:2, labels: app:frontend. 
    - Container: yourusername/yolo-frontend:v2.1.0, port:3000, resources (100m/64Mi requests, 200m/128Mi limits).
- **06-frontend-service.yaml:** 
    - Kind: Service, name: frontend, type: LoadBalancer, selector: app:frontend, port:3000 targets 3000.
- **07-mongo-pdb.yaml:** 
    - Kind: PodDisruptionBudget, name: mongo-pdb, minAvailable:1, selector: app:mongo.
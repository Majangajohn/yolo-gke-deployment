# Implementation Explanation

## 1. Choice of Kubernetes Objects for Deployment
Used Deployments for frontend and backend (stateless, easy scaling). For database, used StatefulSet (extra points) for stable pod identity (mongo-0, mongo-1) and ordered deployment, ideal for storage apps. Replicas=2 across all for fault tolerance—controllers heal by recreating pods. Labels (e.g., app: mongo) track pods; no extra annotations needed.

## 2. Method Used to Expose Pods to Internet Traffic
Frontend: LoadBalancer Service provisions a public IP on port 3000 (targets container 80). Backend and mongo: ClusterIP for internal traffic only (backend on 5000, mongo on 27017). Frontend proxies to backend service.

## 3. Use of Persistent Storage
Yes, via PVC in StatefulSet (1Gi per pod, GKE 'standard' class). Data at /data/db persists across pod deletions/recreations. Replica set ensures data replication between pods.

## 4. Git Workflow
- New repo created.
- Commits: Incremental (e.g., "Add mongo StatefulSet", 10+ total) with descriptive messages.
- Structure: manifests/ folder for YAMLs.
- Pushed to main after each major change.

## 5. Successful Running and Debugging
App runs at README URL: UI loads on port 3000, API calls to backend:5000, DB connects via replica set URI. Cart adds/persists. Debugging: Used `kubectl logs/describe` for issues (e.g., init container errors). Self-healing tested by deleting pods—recreated in seconds, no data loss.

## 6. Good Practices
- Image tags: Versioned (e.g., :1.0) for identification.
- Resources: Requests/limits on all containers for efficient scheduling.
- Replicas=2: Ensures availability.
- Modular manifests for maintainability.
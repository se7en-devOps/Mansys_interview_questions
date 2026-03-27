# Comprehensive Interview Q&A Guide (50 Questions)

## Part 1: Simple Questions (Foundational)

### 1. General Architecture
**Q1: Can you walk me through the end-to-end flow of a user request in your system?**
**A:** A user accesses the React frontend. The request hits the Nginx Ingress Controller on the EC2 instance. Ingress routes the request based on the path—`/` goes to the frontend service, and `/api/analyze` goes to the FastAPI backend. The backend cleans the text, runs it through the BERT model (loaded from S3), and returns a JSON prediction. The frontend then displays this result.

### 2. Tech Stack Selection
**Q2: Why did you choose FastAPI over Flask or Django for this specific use case?**
**A:** I chose FastAPI primarily for its native asynchronous support. My application is I/O bound when saving results to MongoDB and CPU bound during inference. Async allows the server to handle concurrent requests efficiently without blocking. Also, Pydantic validation ensures type safety for the API inputs out of the box, which Flask lacks.

### 3. Containerization
**Q3: Why did you containerize this application instead of just running it on the VM directly?**
**A:** Containerization guarantees consistency across environments. It eliminates "works on my machine" issues by packaging dependencies (Python libraries, Node modules) with the code. It also allows me to use Kubernetes for orchestration, which provides self-healing and easy scaling that raw processes on a VM don't offer.

### 4. Infrastructure choices
**Q4: You used AWS EC2. Why `t3.medium` specifically?**
**A:** The `t3.medium` instance has 2 vCPUs and 4GB of RAM. The BERT model requires significant memory (around 500MB-1GB loaded) and the Kubernetes control plane (Kind) needs overhead. A `t2.micro` (1GB RAM) would essentially crash immediately due to OOM errors. `t3.medium` is the most cost-effective instance that runs the workload stably.

### 5. Kubernetes vs Docker Compose
**Q5: Why use Kubernetes (Kind) instead of just Docker Compose for deployment?**
**A:** Docker Compose is great for local dev, but for production simulation, I wanted Kubernetes features: Zero-downtime rolling updates, Ingress management, and Service Discovery. Kind allows me to mimic a real cluster environment on a single node, which is a closer representation of enterprise deployment than Compose.

### 6. React Frontend
**Q6: Why is the frontend decoupled from the backend? Why not use Jinja templates?**
**A:** Decoupling allows independent scaling and development. The frontend team can iterate on the UI (React) without redeploying the backend API. It also enables me to serve the frontend via a high-performance static server (Nginx) while the Python backend focuses purely on logic.

### 7. Database Choice
**Q7: Why MongoDB? What kind of data are you storing?**
**A:** I'm storing prediction history (User text + AI result). Since the structure of the analysis might change (e.g., adding new metadata fields or confidence scores later), a NoSQL document store like MongoDB offers schema flexibility that a rigid SQL schema wouldn't.

### 8. Model Deployment
**Q8: Where does the BERT model live? Is it in the Git repo?**
**A:** No, never in Git (too large). It lives in an AWS S3 bucket. The application downloads it at startup. This separates code versioning from data/model versioning.

### 9. CI/CD Purpose
**Q9: What is the specific role of GitHub Actions in your pipeline?**
**A:** It handles Continuous Integration. On every push to `main`, it runs unit tests, builds the Docker images for frontend and backend, and pushes them to the GitHub Container Registry (GHCR). It ensures that the artifacts ready for deployment are always clean and buildable.

### 10. GitOps definition
**Q10: Explain "GitOps" to me like I'm a junior engineer.**
**A:** gitOps means "Git is the source of truth." Instead of running `kubectl apply` manually from my laptop, I commit Kubernetes YAML files to Git. A tool inside the cluster (ArgoCD) sees that Git changed and automatically updates the cluster to match. If I change Git, the cluster changes.

---

## Part 2: Medium Questions (Practical Engineering)

### 11. CI vs CD Separation
**Q11: Why do you have GitHub Actions AND ArgoCD? Why not just `kubectl apply` from GitHub Actions?**
**A:** Security and separation of concerns. Giving GitHub Actions input/admin access to my production cluster creates a massive attack surface. With ArgoCD, the cluster reaches *out* to Git (pull model). The cluster credentials never live in the CI provider.

### 12. Model Loading Strategy
**Q12: Downloading the model at startup increases boot time. Why not bake it into the Docker image?**
**A:** Baking it in creates "Fat Containers" (2GB+). pushing/pulling huge images slows down CI/CD pipelines significantly and increases storage costs. Downloading at runtime (with caching or volume mounts) keeps the application image lightweight (<200MB) and agile.

### 13. Terraform vs Ansible
**Q13: You used both Terraform and Ansible. Where exactly do you draw the line between them?**
**A:** Terraform creates the *infrastructure* (VPC, EC2, SG, IAM Roles)—the "hardware". Ansible configures the *software* on that hardware (installing Docker, Kubectl, Kind). I avoid using Terraform `user_data` for complex software installs because Ansible provides better error handling, idempotency, and role management.

### 14. Observability - Latency
**Q14: Users report the application is slow. Which distinct metric in Grafana do you look at first?**
**A:** I look at the 99th percentile (P99) of `prediction_latency_seconds`. Average latency hides outliers. If P99 is high, it means the model inference itself is slow. If P99 is low but users see slowness, the issue might be network ingress or the frontend loading time.

### 15. Ingress Configuration
**Q15: How does NGINX know to send `/api` traffic to Python and `/` to React?**
**A:** It’s defined in the Ingress resource YAML. I use path-based routing rules. One rule matches `path: /api` and directs to the `backend` service (port 8000). The default rule `path: /` directs to the `frontend` service (port 80). Nginx reloads its config automatically when I apply this manifest.

### 16. Failure Handling - Pod Crash
**Q16: If your Python backend crashes due to a bug, what happens? How long is the downtime?**
**A:** Kubernetes restarts the pod automatically (CrashLoopBackOff). If I have multiple replicas (e.g., 3), there is zero downtime because the Service load balancer routes traffic only to healthy pods. If I have only 1 replica, the downtime is the time it takes for the container to restart (~10-20s).

### 17. Secrets Management
**Q17: How does the backend authenticate to MongoDB? Where is the password?**
**A:** The password is injected as an environment variable `MONGODB_URI`. In Kubernetes, this is stored in a `Secret` resource (`kind: Secret`) and referenced in the Deployment YAML. Ideally, I'd use an external secrets manager (like AWS Secrets Manager/Vault), but for this project, K8s Secrets are sufficient.

### 18. Scaling Triggers
**Q18: You decide to auto-scale. What metric would you use to trigger the HPA (Horizontal Pod Autoscaler)?**
**A:** CPU Utilization. BERT inference is computationally heavy. Scaling on memory creates risk (since loading the model consumes fixed memory). Scaling on Request Count is okay, but CPU saturation is the most direct proxy for "I am busy" in this workload.

### 19. Docker Build Optimization
**Q19: How did you optimize your Dockerfiles?**
**A:** I used **Multi-Stage Builds**.
1.  **Builder Stage:** Installs compilers (gcc), downloads dependencies, builds the React app.
2.  **Final Stage:** Copies only the necessary artifacts (the compiled React `build/` folder or installed Python packages) to a slim base image (like `python:3.9-slim` or `nginx:alpine`).
This reduced image size from ~800MB to ~150MB.

### 20. S3 Security
**Q20: Your app downloads from S3. Did you hardcode AWS Access Keys in the code?**
**A:** No. I used **IAM Roles**. Terraform creates an IAM Role with `S3ReadOnly` access and attaches it to the EC2 instance profile. The Python `boto3` library automatically detects these temporary credentials from the instance metadata. No long-lived keys exist in my code.

### 21. ArgoCD Self-Healing
**Q21: What happens if I manually delete a Deployment in the cluster?**
**A:** ArgoCD detects "Drift" (Current state != Git state). Since I have `selfHeal: true` enabled, ArgoCD will almost immediately re-apply the deployment manifest from Git, creating the deployment again. It undoes manual interference.

### 22. Prometheus Scraping
**Q22: How does Prometheus get the data? Does the app push it?**
**A:** Prometheus uses a **Pull Model**. My FastAPI app exposes a `/metrics` endpoint (using `prometheus_client`). Prometheus is configured to scrape (HTTP GET) this endpoint every 15 seconds to collect the current state of counters and gauges.

### 23. Dealing with Dirty Data
**Q23: What if a user submits HTML or JavaScript code in the text box?**
**A:** The Data Cleaning utility in the backend handles this. I use regex/BeautifulSoup to strip HTML tags and special characters before tokenization. Also, Pydantic validates the input type is a string. This prevents XSS payloads from confusing the model (though XSS is primarily a frontend concern, cleaning inputs is best practice).

### 24. Backend Concurrency
**Q24: Python is single-threaded (GIL). How does FastAPI handle multiple requests?**
**A:** I use `async def` for I/O bound tasks (database). For the CPU-bound model inference, FastAPI (via Starlette) runs the endpoint in a **Thread Pool** if it's defined as a standard `def`, or I can explicitly offload the inference to a separate process/thread to avoid blocking the main event loop.

### 25. Terraform State
**Q25: Where is your Terraform State file stored?**
**A:** Currently local (`terraform.tfstate`). In a team environment, I would store it in a remote backend (S3 + DynamoDB for locking) to prevent state corruption when multiple engineers run `terraform apply`.

---

## Part 3: Hard Questions (Senior-Level)

### 26. Architecture Bottlenecks
**Q26: Your system is growing. Where is the absolute first bottleneck you will hit?**
**A:** The Single EC2 Node. No matter how many pods I scale, they share the same physical CPU/RAM of the `t3.medium`. Once the node is saturated, Kubernetes cannot schedule more pods.
*   **Fix:** Use a multi-node cluster (EKS or self-managed) and implement Cluster Autoscaler to add nodes dynamically.

### 27. Model Rollback Strategy
**Q27: You deploy a new model `v2` and accuracy drops to 50%. How do you rollback immediately?**
**A:**
1.  **GitOps:** Revert the commit in Git that changed the `S3_KEY` or image tag.
2.  **ArgoCD:** Syncs the revert.
3.  **K8s:** Starts a rolling update back to the old version.
*   **Better approach (Canary):** Use Argo Rollouts. Deploy `v2` to 5% of traffic. Monitor metrics. If accuracy/error rate is bad, auto-rollback before 100% of users see it.

### 28. Kind vs EKS Trade-offs
**Q28: If money was no object, would you still use Kind? Why move to EKS?**
**A:** No, I'd move to EKS.
*   **Kind Limits:** It's single-node (usually), has no real load balancer integration (uses NodePort/HostPort hacks), and network performance is degraded (overlay within overlay).
*   **EKS Benefits:** Managed control plane (High Availability), native VPC CNI networking (pods get real VPC IPs), and integration with AWS ALB/NLB.

### 29. StatefulSets vs Deployments
**Q29: You used a Deployment for MongoDB (in Docker Compose). If you moved DB to K8s, would you use Deployment?**
**A:** No, I would use a **StatefulSet**. Databases need stable network identities (`db-0`, `db-1`) and stable persistent storage. A Deployment treats pods as ephemeral/interchangeable, which leads to data corruption or split-brain scenarios in distributed databases.

### 30. Zero-Downtime Deployments
**Q30: How do you ensure ZERO downtime during an update? Be specific about probes.**
**A:**
1.  **RollingUpdate Strategy:** `maxUnavailable: 0`, `maxSurge: 1`. K8s starts a new pod before killing an old one.
2.  **Readiness Probe:** This is the key. The new pod is not sent traffic until the `readinessProbe` passes (meaning the model is loaded). The old pod handles traffic until the new one says "I'm ready".
3.  **Graceful Shutdown:** The app must handle `SIGTERM` to finish current requests before dying.

### 31. Cold Starts
**Q31: Your pod takes 30 seconds to download the model. Autoscaling is too slow. How do you fix this?**
**A:**
1.  **InitContainers:** Prefetch model data to a shared volume so the app container starts faster.
2.  **PVC/EFS:** Mount a shared persistent volume containing the model. All pods read from it instantly without downloading.
3.  **Over-provisioning:** Keep a "pause" pod running to hold reserve capacity (PriorityClass).

### 32. Security - Image Scanning
**Q32: How do you know your Docker images don't have vulnerabilities?**
**A:** I would integrate **Trivy** or **Clair** into the Github Actions pipeline. It scans the built image for CVEs (Common Vulnerabilities) in the OS packages and Python libraries. If high-severity CVEs are found, the CI pipeline fails and blocks the push to GHCR.

### 33. Terraform Refactoring
**Q33: You want to separate the VPC creation from the EC2 creation into different state files. Why?**
**A:** Reduces "Blast Radius". Networking (VPC) changes rarely. App servers change often. If they share a state file, a bad `apply` on the server could accidentally mess up the VPC routing. Separating them (layered architecture) enables faster, safer applies.

### 34. Observability - Cardinality
**Q34: You added a metric for `user_id`. Why is this a bad idea in Prometheus?**
**A:** **High Cardinality**. Prometheus creates a new time series for every unique label value. If you have 1 million users, you create 1 million time series. This explodes memory usage and kills the Prometheus server. I should only use low-cardinality labels (status code, endpoint, region).

### 35. Advanced GitOps
**Q35: How do you handle secrets (API keys) in GitOps? You can't commit them to Git.**
**A:**
1.  **Bitnami Sealed Secrets:** Encrypt secrets locally, commit the encrypted fluid to Git. Only the controller in the cluster can decrypt them.
2.  **External Secrets Operator:** Commit a reference (e.g., "get secret from AWS Secrets Manager"). The operator fetches the actual value at runtime.

### 36. Traffic Splitting
**Q36: How would you A/B test a new Model V2?**
**A:** I would use a Service Mesh (like Istio or Linkerd) or Argo Rollouts.
*   **Istio:** Define a `VirtualService` to send 90% traffic to `service-v1` and 10% to `service-v2`.
*   **Result:** I can compare business metrics (user engagement) between the versions before committing.

### 37. Cost Optimization
**Q37: The `t3.medium` is getting expensive. How do we reduce costs without stopping the app?**
**A:**
1.  **Spot Instances:** Use Spot instances for the K8s nodes (saving ~70%). Handle interruptions with graceful termination handling.
2.  **Model Quantization:** Quantize BERT to `int8` (ONNX Runtime). It reduces memory usage by 4x, possibly allowing a downgrade to `t3.small`.

### 38. Ansible Idempotency
**Q38: A junior engineer ran the Ansible playbook 5 times. Did it break anything?**
**A:** It shouldn't, if written correctly. Ansible tasks should be **idempotent**. e.g., "State=present". If the package is installed, do nothing. If I used `shell: wget model`, it would download 5 times (bad). I should use `get_url` or `creates` flag to ensure idempotency.

### 39. Database Migrations
**Q39: You update the data schema. How do you handle DB migrations in this GitOps flow?**
**A:** I would use a **Kubernetes Job** triggered by a comprehensive Helm hook or mapped as a `PreSync` hook in ArgoCD. Before the new code deployment rolls out, the migration Job runs. If it fails, the deployment aborts.

### 40. Thundering Herd
**Q40: The server restarts. 1000 users try to reconnect instantly. What happens?**
**A:** **Thundering Herd problem**. The CPU spikes trying to authenticate/load 1000 requests, causing timeouts, causing users to retry, worsening the load.
*   **Solution:** Implement **Exponential Backoff** with Jitter on the client/frontend. On the backend, implement Rate Limiting (Token Bucket) to shed excess load and survive.

### 41. Logging Architecture
**Q41: Where are the logs? `kubectl logs` is ephemeral.**
**A:** I need centralized logging. I would deploy **Fluentd** or **Promtail** as a DaemonSet. It tails the container logs from the node and ships them to ElasticSearch (ELK stack) or Loki/Grafana. This allows querying historical logs even after pods die.

### 42. CPU Throttling
**Q42: You set CPU limits in K8s. Now latency is spiky. Why?**
**A:** **CFS Throttling**. If I set a hard limit (e.g., `500m`), and the app tries to burst to `700m` for 10ms, the Linux kernel completely pauses the process for the rest of the period.
*   **Fix:** Remove CPU Limits (keep Requests), or ensure Limits are generous enough for bursts.

### 43. Network Policies
**Q43: How do you prevent the Frontend pod from talking directly to the DB (bypassing Backend)?**
**A:** **NetworkPolicies**. By default, K8s is flat (allow-all). I would define a `NetworkPolicy` that says: "MongoDB Ingress: Allow ONLY from Backend Pods". This enforces the architecture at the kernel level.

### 44. Ingress vs LoadBalancer
**Q44: Why use Ingress? Why not just `type: LoadBalancer` for every service?**
**A:** Cost and IP exhaustion. `LoadBalancer` creates a physical AWS ELB ($$$) for *every* service. Ingress allows me to use ONE LoadBalancer (one IP) to route traffic to 50 different microservices based on Host/Path.

### 45. Terraform Modules
**Q45: Your Terraform file is 500 lines. How do you clean it up?**
**A:** Refactor into **Modules**. Create a `modules/vpc`, `modules/ec2`, `modules/security` folder structure. The `main.tf` then essentially just calls these modules with variables. Enhances reusability and readability.

### 46. Disaster Recovery
**Q46: AWS `ap-south-1` Region goes down completely. How fast can you recover?**
**A:** Since I have Infrastructure as Code: 
1.  Change `region = "us-east-1"` in Terraform variables.
2.  `terraform apply`.
3.  Update Ansible inventory.
4.  Run Ansible.
5.  Wait for ArgoCD to sync.
*   **RTO (Recovery Time Objective):** ~15-20 minutes. The only loss is the MongoDB data if I didn't verify cross-region backups.

### 47. API Gateway Pattern
**Q47: Why did you simple use Nginx Ingress? Why not AWS API Gateway?**
**A:** Avoid Vendor Lock-in and cost. Nginx Ingress is standard K8s. It works locally (Kind), on AWS, on Google Cloud. AWS API Gateway ties me to AWS and charges per request. For this scale, Nginx on the cluster is simpler and free.

### 48. Distributed Tracing
**Q48: A request takes 2s. You don't know if it's the DB or BERT. How do you find out?**
**A:** I need **Distributed Tracing** (Jaeger/OpenTelemetry). I would instrument the FastAPI code to create spans: "Span A: Prediction", "Span B: Mongo Save". Jaeger visualizes this waterfall, showing exactly where the time was spent.

### 49. MLOps Drift
**Q49: The model was accurate last month, but inaccurate today. Code didn't change. What happened?**
**A:** **Data Drift**. The real-world news topics changed (e.g., a new war or election), but the model was trained on old data.
*   **Solution:** I need a retraining pipeline. Monitor "Prediction Distribution". If it skews, trigger a workflow to retrain BERT on newer datasets.

### 50. Final question
**Q50: Project Retrospective. If you could delete one tool from your stack to simplify, which one?**
**A:** **Ansible**.
*   **Why:** For a single immutable server, I could use **Packer** to build an AMI with Docker/Kind pre-installed. Then Terraform just deploys that AMI. It removes the configuration step after provisioning, making the spin-up faster and purely immutable.

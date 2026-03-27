## 5️⃣ Architecture & System Flow (7 Questions)

**34. Walk me through the end-to-end data flow from a user submitting text to receiving a prediction.**
*Strong Answer:* Client sends a `POST` request with JSON payload (`{"text": "breaking news..."}`). The API Gateway/Ingress routes it to the FastAPI pod. Pydantic validates the schema. The text is passed to the loaded preprocessing function (regex/lemmatization), then `.transform()` converts it via the serialized TF-IDF vectorizer into a sparse matrix. This matrix is passed into the loaded Logistic Regression `.predict_proba()` method. A 0.0-1.0 float is returned, formatted into JSON (`{"is_fake": true, "confidence": 0.88}`), and sent back to the client.
*Business Reasoning:* Understanding the critical path allows us to identify and fix latency bottlenecks.
*Technical Explanation:* The ML model and vectorizer are loaded *once* during application startup (`@app.on_event("startup")`) into global memory. Hitting the disk or S3 for every request is a junior mistake that destroys throughput.
*Trade-offs:* Loading models into memory consumes 50-500MB per worker, limiting the number of parallel uvicorn workers we can pack onto a cheap EC2 instance.

**35. Where exactly does the preprocessing run?**
*Strong Answer:* Specifically within the FastAPI Python application, bound tightly to the inference logic.
*Business Reasoning:* Consistency is paramount. The exact same data transformations used during training must be identical in production.
*Technical Explanation:* Ideally, the preprocessing is wrapped in a custom scikit-learn `TransformerMixin` and serialized as part of the total Pipeline `.pkl` file. If not possible (e.g., relying on external SpaCy models), the preprocessing utility functions are imported directly into the API route.
*Trade-offs:* Coupling preprocessing logic directly into the API microservice makes it monolithic and difficult to share with other ML services (e.g., a summarizer) that might need the exact same cleaned text.

**36. How and where is the trained model stored?**
*Strong Answer:* The finalized artifact (`model_v1.joblib`) is uploaded to an AWS S3 bucket (or local persistent `.volumes/` if local Docker). The API pulls this artifact down during the Docker `build` phase or application startup.
*Business Reasoning:* Version control of binaries is required for auditing and rollback capability when a model degrades. Git is for code, not 200MB model files.
*Technical Explanation:* `joblib` is vastly superior to `pickle` for scikit-learn objects because it is optimized for fast I/O on large numpy arrays. A registry (like MLflow) tracks the S3 URI associated with the best training run.
*Trade-offs:* Pulling models from S3 at runtime increases container cold-start time. Baking the model into the Docker image directly makes the image massive but boot times extremely fast.

**37. Describe your API Architecture.**
*Strong Answer:* A synchronous RESTful API built with FastAPI and Uvicorn.
*Business Reasoning:* It's the industry standard for fast Python microservices due to native async support (though ML inference is CPU-bound and therefore synchronous).
*Technical Explanation:* The endpoint (`/analyze`) is defined with Pydantic models for strict In/Out type coercion. It operates synchronously because `.predict()` blocks the CPU. Hiding synchronous ML code behind an `async def` wrapper in FastAPI actually *hurts* performance because it sends CPU-heavy workloads to a limited threadpool designed for I/O waits.
*Trade-offs:* gRPC would be significantly faster for internal microservice communication due to Protobuf binary serialization, but REST/JSON is universally compatible for external web clients.

**38. Batch processing versus Real-time API: Which is better for this project?**
*Strong Answer:* Real-time REST API is mandatory for moderation (preventing publication). Batch processing (Kafka/Airflow) is superior for historical auditing (sweeping the DB for fake news published yesterday).
*Business Reasoning:* The architecture depends entirely on the product use case.
*Technical Explanation:* Real-time requires 24/7 provisioned compute waiting for requests (expensive). Batch processing allows us to spin up cheap Spot Instances, predict 1 million stored rows, and shut down (cheap).
*Trade-offs:* High latency vs. High infrastructure cost.

**39. Where are the latency bottlenecks in your system?**
*Strong Answer:* It is rarely the ML `.predict()` method (LogReg takes ~10 microseconds). The bottleneck is text preprocessing (tokenization/lemmatization) and HTTP overhead.
*Business Reasoning:* Engineers who optimize the wrong thing waste money.
*Technical Explanation:* Running a full NLP pipeline (SpaCy) to lemmatize 1,000 words takes 100-200 milliseconds. This completely dominates the total API response time.
*Trade-offs:* Dropping lemmatization and relying purely on TF-IDF n-grams reduces accuracy slightly but guarantees sub-20ms latency.

**40. What is your Scaling strategy if the API goes viral?**
*Strong Answer:* Horizontal scaling using Kubernetes (kind/EKS).
*Business Reasoning:* We need elasticity—scaling up during breaking news events and scaling down to zero at 3 AM to save costs.
*Technical Explanation:* Because the FastAPI container is completely stateless (the model is in read-only memory), we can spin up 100 replicas behind an Nginx Ingress Controller or AWS ALB. The bottleneck becomes network bandwidth, not compute.
*Trade-offs:* Distributed state (like updating a rate-limiting Redis cache across 100 pods) introduces significant architectural complexity.

## 6️⃣ Performance, Scaling & Deployment (5 Questions)

**41. How would you handle 1,000,000 predictions per day?**
*Strong Answer:* 1M/day is roughly 12 requests per second (RPS). A single threaded uvicorn worker running a TF-IDF LogReg model can efficiently handle 50-100 RPS. This workload technically doesn't even require Kubernetes; a single $5 EC2 instance running Docker Compose handles it easily.
*Business Reasoning:* Over-engineering infrastructure for small workloads wastes developer time.
*Technical Explanation:* 1M rows of text data (average 500 words) is maybe 3-5 GBs of raw data. This fits comfortably in the RAM of a standard modern server if processed in moderate batches.
*Trade-offs:* A single EC2 instance is a Single Point of Failure (SPOF). We sacrifice reliability for extreme cost savings.

**42. How are you handling Model Versioning?**
*Strong Answer:* Strictly decoupling the model artifact version from the Git code version.
*Business Reasoning:* The code (API) might not change, but the model (data) must update weekly to catch new fake narratives.
*Technical Explanation:* I use semantic versioning (v1.2.0) for the model binaries stored in S3, and pass that version string into the Docker container via an environment variable (`MODEL_VERSION=v1.2.0`). The API logs this version in the JSON response so data science teams can trace exactly which model made a bad prediction.
*Trade-offs:* Requires a dedicated tracking system (MLflow or Weights & Biases) to link the `v1.2.0` string back to the exact training dataset and hyperparameter run.

**43. How do you detect Model Drift in production?**
*Strong Answer:* By continuously monitoring the distribution of prediction confidence scores.
*Business Reasoning:* Fake news evolves. If we don't detect when the model becomes obsolete, we expose the business to massive liability.
*Technical Explanation:* I expose Prometheus metrics from the FastAPI app visualizing the ratio of Fake/Real predictions over a 24-hour window. If the model historically flags 10% of traffic, but suddenly flags 40%, the data distribution has shifted (covariate shift), or the model is broken.
*Trade-offs:* Statistical drift detection is difficult to tune. A sudden spike might be actual breaking fake news (a real event), not model degradation.

**44. Explain CI/CD for Machine Learning (CT - Continuous Training).**
*Strong Answer:* It's not just testing code, it’s testing data validity and model performance automatically.
*Business Reasoning:* Human retraining is perfectly manual and prone to error. Automation guarantees deployment speed and consistency.
*Technical Explanation:* GitHub Actions triggers on push. It runs `pytest` for the FastAPI logic. Then, an Airflow DAG pulls the latest data, runs the `train.py` script, asserts that the new F1 score is >= the production F1 score, serializes the artifact, builds a new Docker image containing `model_latest`, and applies a `kubectl rollout restart` to the cluster.
*Trade-offs:* Fully automated ML deployment is highly dangerous without human "shadow mode" validation to catch edge cases the automated tests missed.

**45. What are the constraints of Cloud Deployment for this specific project?**
*Strong Answer:* Memory limits and bandwidth egress costs.
*Business Reasoning:* Cloud providers overcharge for RAM and out-bound network traffic.
*Technical Explanation:* If we switch from TF-IDF to a 1.5GB Transformer model, the Kubernetes pods will constantly crash with `OOMKilled` errors unless we drastically provision more expensive, RAM-heavy worker nodes. Sending millions of JSON responses out of AWS costs actual money ($0.09 per GB).
*Trade-offs:* Using highly compressed libraries and optimizing JSON payloads saves infrastructure costs but increases raw developer engineering time.

## 7️⃣ Failure & Edge Cases (5 Questions)

**46. How does your model handle Adversarial Attacks (e.g., typos purposefully inserted to bypass the filter)?**
*Strong Answer:* TF-IDF fails completely. If a bad actor writes "V!rUs", it’s an Out-Of-Vocabulary word and bypassed entirely.
*Business Reasoning:* Malicious actors actively try to defeat moderation algorithms.
*Technical Explanation:* To defend against this, we use Character-level N-grams instead of Word-level, or deploy a subword spelling corrector (like SymSpell) *before* the vectorizer. Transformers natively handle this better using sub-word tokenization (Byte-Pair Encoding).
*Trade-offs:* Subword tokenization and spelling correction add significant latency.

**47. How do you classify Sarcasm or Satire (e.g., The Onion)?**
*Strong Answer:* I fundamentally separate the problem. Sarcasm detection requires syntax and tone analysis, which Bag-of-Words TF-IDF completely destroys.
*Business Reasoning:* Banning satirical news causes massive PR backlash and alienates users.
*Technical Explanation:* We handle satire through explicit declarative allow-listing (whitelisting URLs like `theonion.com`) before the ML model even sees the text. ML is terrible at satire without massive, specific fine-tuning.
*Trade-offs:* Hardcoding rules (allow/deny lists) is brittle and requires manual maintenance, but it is 100% accurate and instant.

**48. What if an article is 80% real facts, but the final paragraph is a fabricated lie?**
*Strong Answer:* An aggregate document-level model will likely fail and predict "Real" because the dominant TF-IDF vectors point to truth.
*Business Reasoning:* Disinformation often hides inside verified facts to gain credibility.
*Technical Explanation:* The architectural solution is to chunk the document. Instead of predicting on the whole text, we split the article into paragraph chunks, run predictions on each chunk, and use a Max Pooling aggregation (if *any* chunk > 0.9 Fake, flag the whole article).
*Trade-offs:* Chunking multiplies the inference cost by the number of chunks (e.g., 5 paragraphs = 5x the compute cost per article).

**49. What is Concept Drift versus Data Drift?**
*Strong Answer:* Data drift is when the language changes (e.g., people start talking about a new API). Concept drift is when the underlying truth changes (e.g., "Masks don't work" was considered true early in the pandemic, then false later).
*Business Reasoning:* Models do not understand "truth," they understand patterns. Factuality requires external grounding.
*Technical Explanation:* An NLP classifier cannot solve Concept Drift. It requires a Retrieval-Augmented Generation (RAG) architecture or Knowledge Graph lookup to verify statements against an external database of facts (Wikidata).
*Trade-offs:* RAG architectures are 10x more complex, infinitely slower, and require expensive vector databases.

**50. How did you ensure your predictions weren't Biased against specific political keywords?**
*Strong Answer:* Regularization and explicit feature ablation.
*Business Reasoning:* We must prove our algorithm only detects deceptive communication styles, not censorship of specific conservative/liberal names.
*Technical Explanation:* I used L1 Regularization to force the model to drop features. Then I analyzed the remaining non-zero coefficients. If "Trump" or "Biden" had high predictive weights for "Fake", my dataset was fundamentally biased. The mitigation is masking those entities with `[PERSON]` during preprocessing so the model is forced to rely on verbs and adjectives (e.g., "shocking secret revealed").
*Trade-offs:* Entity masking prevents the model from understanding bad actors entirely. It makes the model fairer but mathematically weaker.

# SECTION 3 — Advanced Technical Deep Dive

*(Provide this document to prove senior-level mathematical intuition).*

### 1. The Mathematics of Logistic Regression for Text Classification
Logistic Regression doesn't output classes; it outputs probabilities. It uses the linear combination of word weights ($w$) and word frequencies ($x$), and maps them between 0 and 1 using the Sigmoid curve.
$$P(y=1 | x) = \frac{1}{1 + e^{-(w^T x + b)}}$$
*Why it works on text:* Because $x$ is a TF-IDF sparse matrix, $w^T x$ is incredibly fast to compute via sparse dot products. Each word in the vocabulary gets a coefficient ($w$). If $w_{clickbait} = +3.5$ and $w_{reuters} = -2.1$, the presence of those words pushes the linear sum left or right along the S-curve.

### 2. TF-IDF Formula Breakdown
**Term Frequency (TF):** $TF(t, d) = \frac{count(t \in d)}{Length(d)}$ (How often the word appears in *this* document).
**Inverse Document Frequency (IDF):** $IDF(t, D) = \log(\frac{N}{df(t)})$
*Where:* $N$ = total documents, $df$ = number of docs containing term $t$.
*The Magic:* If 'the' is in 10,000 out of 10,000 docs, $10k/10k = 1$. $\log(1) = 0$. The IDF is 0, completely neutralizing the word's impact, acting as an automatic stopword filter.

### 3. Precision, Recall, and F1-Score Formulas
**Precision:** ($TP / (TP + FP)$) - Of all the articles I *flagged*, how many were actually fake? (Punishes False Alarms).
**Recall:** ($TP / (TP + FN)$) - Of all the *actual* fake articles out there, how many did I catch? (Punishes Missing things).
**F1-Score:** $2 \times \frac{Precision \times Recall}{Precision + Recall}$ (The Harmonic Mean. If either drops to zero, the whole score crashes).

### 4. Confusion Matrix Deep Dive
For a production ML system, the quadrants mean specific business outcomes:
*   **True Positive (TP):** Caught Fake News. (Success).
*   **True Negative (TN):** Ignored Real News. (Success).
*   **False Positive (FP):** Flagged a legitimate journalist. (Massive PR disaster; angry users).
*   **False Negative (FN):** Let Russian disinformation slip onto the feed. (Slow brand decay).

### 5. ROC Curve Explanation
The Receiver Operating Characteristic curve maps True Positive Rate (Recall) on the Y-axis against False Positive Rate on the X-axis. A perfect model shoots straight up to (0,1). A random guess model is a diagonal line from (0,0) to (1,1). The Area Under Curve (ROC-AUC) quantifies this. An AUC of 0.5 is useless. An AUC of 0.95 means the model is highly confident in separating the distributions.

### 6. Bias-Variance Tradeoff
**High Bias (Underfitting):** The model is too simple (e.g., Logistic Regression with massive L2 penalty). It cannot learn the complex patterns of fake news. Training F1 is bad, Validation F1 is bad.
**High Variance (Overfitting):** The model is too complex (e.g., Deep Decision Tree). It memorized the training set perfectly. Training F1 is 99%, Validation F1 is 70%.
**The Solution:** Cross-validation helps visualize this, and hyperparameter tuning (adjusting model complexity and regularization constraints) finds the optimal middle ground.

### 7. Why Text Classification Suffers from "Sparsity" (The Curse of Dimensionality)
A corpus with 50,000 unique words creates a 50,000-dimensional matrix. One document might only contain 300 words. Thus, 49,700 columns for that row are explicitly $0.0$.
*The Problem:* Distances between vectors in 50k-dimensional space become almost uniform (Euclidean distance breaks down). Algorithms like KNN completely fail here. Linear algorithms (SVM/LogReg) thrive because they look for hyperplanes to slice the empty space, rather than relying on point-to-point proximity.

### 8. How Transformers (BERT) Improve "Context Understanding"
TF-IDF is Bag-of-Words; word order is irrelevant.
Transformers use **Self-Attention**. When processing the word "bank", a Transformer mathematically calculates the relationship/relevance of "bank" to every other word in the sequence simultaneously via attention weights (e.g., associating "bank" strongly with "river" vs. "money"). This natively handles polysemy (words with multiple meanings) and long-range sentence dependencies, which TF-IDF cannot accomplish natively.

# 🐳 AWS ECR Credentials Auto-Refresh in Kubernetes

## 📖 Overview
AWS Elastic Container Registry (ECR) provides short-lived authentication tokens that expire **every 12 hours**.  
Kubernetes uses these tokens (via an ImagePullSecret) to pull container images from ECR.  

When these credentials expire, pods fail to pull new images — resulting in image pull errors such as:
```
Failed to pull image: ... authorization token has expired
```

To solve this, we created an automated mechanism that **refreshes the ECR credentials every 11 hours** inside the Kubernetes cluster.

---

## ⚙️ How It Works
The process is handled through a **Kubernetes CronJob** that runs periodically and:
1. Authenticates to AWS using stored IAM credentials (from a Kubernetes Secret).  
2. Deletes the existing ECR image pull secret (`ecr-creds`).  
3. Regenerates a new Docker registry secret with a fresh ECR token.  
4. Ensures that all deployments using that secret can continue pulling images without interruption.

---

## 🧩 Components Breakdown

### 1. **Service Account**
File: `cronjob-servic-account.yaml`

This ServiceAccount (`cronjob-sa`) allows the CronJob to interact with the Kubernetes API — specifically to create and delete secrets in the target namespace (`nextplane-staging`).

---

### 2. **Role**
File: `cronjob-role.yaml`

This Role defines the **permissions** needed by the CronJob:
- `create`, `get`, and `delete` permissions for `secrets`
- Scoped to the namespace (`nextplane-staging`)

---

### 3. **RoleBinding**
File: `cronjob-rolebinding.yaml`

This binds the **Role** to the **ServiceAccount**, granting it the defined permissions.  
Without this binding, the CronJob wouldn’t have access to manage Kubernetes secrets.

---

### 4. **AWS Credentials Secret**
Manually created using:
```bash
kubectl create secret generic aws-credentials   -n nextplane-staging   --from-literal=aws_access_key_id=<ACCESS_KEY>   --from-literal=aws_secret_access_key=<SECRET_KEY>   --from-literal=aws_session_token=<SESSION_TOKEN>
```

This secret provides temporary AWS credentials used by the CronJob to authenticate to ECR and request new tokens.

---

### 5. **CronJob**
File: `cronjob.yaml`

This is the heart of the setup.  
It runs every **11 hours** (`0 */11 * * *`) and performs these actions:

1. Logs in to AWS ECR using the AWS CLI  
2. Deletes the old ECR image pull secret  
3. Creates a new one with an updated authentication token  
4. Verifies the operation and logs success

**Command snippet inside the CronJob:**
```bash
aws ecr get-login-password --region us-east-1
kubectl delete secret -n nextplane-staging --ignore-not-found ${SECRET_NAME}
kubectl create secret -n nextplane-staging docker-registry ${SECRET_NAME}   --docker-server=921060721169.dkr.ecr.us-east-1.amazonaws.com   --docker-username=AWS   --docker-password="$(aws ecr get-login-password --region us-east-1)"
```

---

## 🧠 Why This Is Needed
- **AWS ECR tokens expire every 12 hours.**  
- If not refreshed, Kubernetes can no longer authenticate with ECR.
- The **CronJob** ensures continuous access by regenerating the secret automatically.
- This removes the need for manual secret updates and avoids service downtime.

---

## 🧾 Deployment Steps

1. **Apply the Service Account**
   ```bash
   kubectl apply -f cronjob-servic-account.yaml
   ```

2. **Apply the Role**
   ```bash
   kubectl apply -f cronjob-role.yaml
   ```

3. **Apply the RoleBinding**
   ```bash
   kubectl apply -f cronjob-rolebinding.yaml
   ```

4. **Create the AWS Credentials Secret**
   ```bash
   kubectl create secret generic aws-credentials      -n nextplane-staging      --from-literal=aws_access_key_id=<ACCESS_KEY>      --from-literal=aws_secret_access_key=<SECRET_KEY>      --from-literal=aws_session_token=<SESSION_TOKEN>
   ```

5. **Deploy the CronJob**
   ```bash
   kubectl apply -f cronjob.yaml
   ```

6. **Verify**
   ```bash
   kubectl get cronjob -n nextplane-staging
   kubectl get pods -n nextplane-staging
   ```

You should see a new job running every 11 hours automatically.

---

## 🔍 Manual Testing (Optional)

If you want to test the logic manually, you can convert the CronJob into a **Pod** and execute commands interactively:
```bash
kubectl apply -f pod-test.yaml
kubectl exec -it ecr-creds-refresh-test -n nextplane-staging -- sh
```

---

## ✅ Expected Outcome
- The ECR image pull secret (`ecr-creds`) is recreated every 11 hours.
- Kubernetes continues pulling images without authentication errors.
- No manual intervention is required once configured.

---

## 🧰 Tools Used
- **AWS CLI**
- **kubectl**
- **Kubernetes CronJob**
- **IAM Credentials (temporary session token)**

---

## 👨‍💻 Maintainer
**Author:** Muhammad Umer  
**Organization:** Teammate Technology  
**Environment:** `nextplane-staging`  
**Region:** `us-east-1`

---

## 🕒 Schedule Summary
| Task | Frequency | Description |
|------|------------|-------------|
| ECR token refresh | Every 11 hours | Refreshes ECR login credentials before expiration |

---

## 🚀 Notes
- The CronJob uses **temporary AWS session credentials**.  
  Ensure they are rotated if you’re not using an IAM Role with automatic refresh.
- Adjust the `--region` or secret name as needed for other environments.

---

**This setup ensures that all your Kubernetes workloads can seamlessly pull images from AWS ECR without ever hitting authentication expiration errors.**

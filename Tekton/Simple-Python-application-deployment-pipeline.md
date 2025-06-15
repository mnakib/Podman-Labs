## Creating a Hello World Python image using Tekton

Below steps are an example of deploying a Hello World Python application. For this, we are using a Tekton Pipeline that accomplishes the following, which are the core CI/CD steps for a Python application:

1.  **Clones a Git Repository** (containing your Python "Hello World" app).
2.  **Builds a Docker Image** for the Python application.
3.  **Pushes the Docker Image to Docker Hub**.

This example will assume you have:

* A **simple Python "Hello World" application** in a Git repository.
* A **Dockerfile** in the root of that repository.
* **Docker Hub credentials** stored as a Kubernetes Secret.

---

### **Assumptions:**

* **Git Repository:** `https://github.com/your-username/python-hello-world.git`
* **Python App (e.g., `app.py`):**
    ```python
    # app.py
    print("Hello, Tekton Python App!")
    ```
* **Dockerfile:**
    ```dockerfile
    # Dockerfile
    FROM python:3.9-slim-buster
    WORKDIR /app
    COPY requirements.txt .
    RUN pip install -r requirements.txt
    COPY . .
    CMD ["python", "app.py"]
    ```
    (Even for "Hello World," it's good practice to have `requirements.txt`, even if empty for now)
* **`requirements.txt`:** (can be empty, or if you add libraries later)
    ```
    # requirements.txt
    ```
* **Docker Hub Username:** `your-dockerhub-username`
* **Docker Hub Image Name:** `python-hello-world-app`
* **Kubernetes Secret for Docker Hub:** A secret named `dockerhub-secret` in the `default` namespace (or your preferred namespace) containing your Docker Hub username and password. You can create it like this:
    ```bash
    kubectl create secret docker-registry dockerhub-secret \
      --docker-server=docker.io \
      --docker-username=<your-dockerhub-username> \
      --docker-password=<your-dockerhub-password> \
      --namespace=default # Or your namespace
    ```

---

### **Tekton Components:**

We'll need the following Tekton resources:

1.  **`Task` - `git-clone-task`**: To clone the repository.
2.  **`Task` - `build-and-push-image-task`**: To build the Docker image and push it to Docker Hub.
3.  **`Pipeline` - `python-app-pipeline`**: Orchestrates the `git-clone-task` and `build-and-push-image-task`.
4.  **`PipelineRun` - `python-app-pipeline-run`**: Triggers an execution of the `python-app-pipeline`.
5.  **`PersistentVolumeClaim` (PVC) - `workspace-pvc`**: To provide shared storage for the pipeline.

---

### **YAML Files:**

**1. `01-git-clone-task.yaml`**

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: git-clone-task
spec:
  params:
    - name: repo-url
      description: The URL of the Git repository to clone.
      type: string
    - name: revision
      description: The Git revision (branch, tag, or commit hash) to checkout.
      type: string
      default: main
  workspaces:
    - name: output
      description: The workspace where the repository will be cloned.
  steps:
    - name: clone-repo
      image: alpine/git:latest
      script: |
        #!/bin/ash
        echo "Cloning repository: $(params.repo-url) at revision $(params.revision) into $(workspaces.output.path)"
        git clone --depth=1 --branch $(params.revision) $(params.repo-url) $(workspaces.output.path)
        echo "Repository cloned. Listing contents:"
        cd $(workspaces.output.path)
        ls -la
```

**2. `02-build-and-push-image-task.yaml`**

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build-and-push-image-task
spec:
  params:
    - name: image-name
      description: The name of the Docker image to build and push (e.g., your-dockerhub-username/app-name).
      type: string
    - name: context
      description: The build context for the Dockerfile.
      type: string
      default: .
    - name: dockerfile
      description: The path to the Dockerfile to use.
      type: string
      default: Dockerfile
    - name: registry-secret
      description: Name of the Kubernetes secret containing Docker registry credentials.
      type: string
      default: dockerhub-secret # Default to the secret created earlier
  workspaces:
    - name: source
      description: The workspace containing the source code.
  steps:
    - name: build-image
      image: gcr.io/kaniko-project/executor:latest # Kaniko is used to build images in Kubernetes without a Docker daemon
      env:
        - name: DOCKER_CONFIG
          value: /tekton/home/.docker
      args:
        - --dockerfile=$(params.dockerfile)
        - --context=$(workspaces.source.path)/$(params.context)
        - --destination=$(params.image-name):latest
        - --destination=$(params.image-name):$(results.commit-sha.url) # Tag with commit SHA if desired
        - --skip-tls-verify=false # Set to true if you are using an insecure registry
        - --single-snapshot
      securityContext:
        runAsUser: 0 # Kaniko often needs to run as root

    - name: extract-commit-sha
      image: alpine/git:latest
      script: |
        #!/bin/ash
        cd $(workspaces.source.path)
        COMMIT_SHA=$(git rev-parse --short HEAD)
        echo "Commit SHA: $COMMIT_SHA"
        echo -n "$COMMIT_SHA" > $(results.commit-sha.path)
      results:
        - name: commit-sha
          description: Short Git commit SHA of the built image.

  volumes:
    - name: docker-config
      secret:
        secretName: $(params.registry-secret) # Mount the Docker Hub secret
```

**3. `03-python-app-pipeline.yaml`**

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: python-app-pipeline
spec:
  params:
    - name: repo-url
      description: The URL of the Git repository for the Python app.
      type: string
    - name: repo-revision
      description: The Git revision (branch, tag, or commit hash) for the app.
      type: string
      default: main
    - name: docker-image-name
      description: The full name of the Docker image to build (e.g., your-dockerhub-username/app-name).
      type: string
  workspaces:
    - name: shared-workspace
      description: Workspace for sharing source code between tasks.
  tasks:
    - name: fetch-source
      taskRef:
        name: git-clone-task
      params:
        - name: repo-url
          value: $(params.repo-url)
        - name: revision
          value: $(params.repo-revision)
      workspaces:
        - name: output
          workspace: shared-workspace

    - name: build-and-push
      taskRef:
        name: build-and-push-image-task
      runAfter:
        - fetch-source # This task runs after fetch-source completes
      params:
        - name: image-name
          value: $(params.docker-image-name)
      workspaces:
        - name: source
          workspace: shared-workspace
```

**4. `04-workspace-pvc.yaml`**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: python-app-workspace-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi # Adjust storage as needed
```

**5. `05-python-app-pipeline-run.yaml`**

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: run-python-app-pipeline
spec:
  pipelineRef:
    name: python-app-pipeline
  params:
    - name: repo-url
      value: "https://github.com/your-username/python-hello-world.git" # IMPORTANT: Change this to your actual repo URL
    - name: repo-revision
      value: "main"
    - name: docker-image-name
      value: "your-dockerhub-username/python-hello-world-app" # IMPORTANT: Change this to your Docker Hub username and desired image name
  workspaces:
    - name: shared-workspace
      persistentVolumeClaim:
        claimName: python-app-workspace-pvc
```

---

### **Deployment Steps:**

1.  **Create your Python "Hello World" Git repository and push the `app.py`, `Dockerfile`, and `requirements.txt` files.**
    * Make sure `your-username` and `python-hello-world.git` are correctly set up.
2.  **Create the Docker Hub Secret:**
    ```bash
    kubectl create secret docker-registry dockerhub-secret \
      --docker-server=docker.io \
      --docker-username=<your-dockerhub-username> \
      --docker-password=<your-dockerhub-password> \
      --namespace=default
    ```
3.  **Apply the Tekton Tasks and Pipeline:**
    ```bash
    kubectl apply -f 01-git-clone-task.yaml
    kubectl apply -f 02-build-and-push-image-task.yaml
    kubectl apply -f 03-python-app-pipeline.yaml
    kubectl apply -f 04-workspace-pvc.yaml
    ```
4.  **Run the Pipeline:**
    * **IMPORTANT:** Before running, edit `05-python-app-pipeline-run.yaml` and replace `"https://github.com/your-username/python-hello-world.git"` and `"your-dockerhub-username/python-hello-world-app"` with your actual values.
    ```bash
    kubectl apply -f 05-python-app-pipeline-run.yaml
    ```
5.  **Monitor the PipelineRun:**
    ```bash
    kubectl get pipelineruns -w
    ```
    Wait until the `PipelineRun` shows `Succeeded`.
6.  **Check the logs:**
    ```bash
    tkn pr logs run-python-app-pipeline -f # Using Tekton CLI
    # OR using kubectl
    kubectl logs -f pod/$(kubectl get pod -l tekton.dev/pipelineRun=run-python-app-pipeline,tekton.dev/task=build-and-push -o jsonpath='{.items[0].metadata.name}') -c build-image
    ```
    You should see output from Kaniko showing the image being built and pushed.
7.  **Verify on Docker Hub:** Log in to your Docker Hub account and check if the `your-dockerhub-username/python-hello-world-app` image has been pushed successfully.

---

### **What's Next (Deployment)?**

To "deploy" this application, you would typically:

1.  **Create a Kubernetes Deployment (or similar):**
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: python-hello-app
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: python-hello
      template:
        metadata:
          labels:
            app: python-hello
        spec:
          containers:
            - name: hello-world
              image: your-dockerhub-username/python-hello-world-app:latest # Or the specific commit SHA tag
              ports:
                - containerPort: 80
    ```
    (Note: Your Python app would need to listen on a port if it's a web service; for a simple script, it just prints and exits.)

2.  **Create a Kubernetes Service (if it's a web app):**
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: python-hello-service
    spec:
      selector:
        app: python-hello
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
      type: LoadBalancer # Or ClusterIP, NodePort
    ```

You could extend your Tekton Pipeline to apply these Kubernetes manifests after pushing the image, using a `kubectl` task.

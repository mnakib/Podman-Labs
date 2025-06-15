## What is Tekton?

Tekton helps you define your CI/CD pipelines as Kubernetes resources. Let's walk through a basic example that demonstrates the core concepts: **Tasks** and **Pipelines**.

For this example, you'll need:

1.  **A Kubernetes cluster:** You can use Minikube, Kind, Docker Desktop with Kubernetes enabled, or any cloud Kubernetes service (GKE, EKS, AKS, OpenShift).
2.  **`kubectl`:** The Kubernetes command-line tool.
3.  **`tkn` (Tekton CLI):** The Tekton command-line interface, which simplifies interacting with Tekton resources. You can find installation instructions on the official Tekton documentation.

---

### Step 1: Install Tekton Pipelines

First, install Tekton Pipelines on your Kubernetes cluster.

```bash
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```

Verify that the Tekton components are running:

```bash
kubectl get pods --namespace tekton-pipelines --watch
```

Wait until all pods show a `Running` or `Completed` status. Press `Ctrl+C` to exit the watch.

---

### Step 2: Define a Simple Task

A **Task** is the fundamental building block in Tekton. It defines a series of **steps** that run sequentially, typically inside a container.

Let's create a Task that simply echoes a message.

Create a file named `echo-task.yaml`:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: echo-hello-world
spec:
  steps:
    - name: say-hello
      image: ubuntu  # Using a simple Ubuntu image
      script: |
        #!/bin/bash
        echo "Hello from Tekton!"
```

Apply the Task to your cluster:

```bash
kubectl apply -f echo-task.yaml
```

Verify the Task is created:

```bash
kubectl get tasks
```

You should see `echo-hello-world` in the list.

---

### Step 3: Run the Task using a TaskRun

A **TaskRun** is an instantiation of a Task. It tells Tekton to execute a specific Task.

Create a file named `echo-taskrun.yaml`:

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: echo-hello-world-run
spec:
  taskRef:
    name: echo-hello-world # References the Task we just created
```

Apply the TaskRun to your cluster:

```bash
kubectl apply -f echo-taskrun.yaml
```

Now, let's watch the logs of your TaskRun using the `tkn` CLI:

```bash
tkn taskrun logs echo-hello-world-run -f
```

You should see output similar to this:

```
[say-hello] Hello from Tekton!
```

This confirms your first Tekton Task executed successfully!

---

### Step 4: Define a Pipeline with Multiple Tasks

A **Pipeline** is a collection of Tasks arranged in a specific order. Tasks can pass outputs to subsequent Tasks, creating a workflow.

Let's create two Tasks: one to clone a Git repository and another to list its contents. Then we'll combine them into a Pipeline.

**Task 1: `git-clone-task.yaml`** (This task is commonly available in the Tekton Catalog, but we'll define it simply here for demonstration)

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: git-clone-task
spec:
  params:
    - name: repoUrl
      description: The URL of the Git repository to clone
      type: string
    - name: branch
      description: The Git branch to clone
      type: string
      default: main
  workspaces:
    - name: output
      description: The workspace where the repository will be cloned
  steps:
    - name: clone
      image: alpine/git # A small image with git installed
      script: |
        #!/bin/ash
        git clone "$(params.repoUrl)" "$(workspaces.output.path)" -b "$(params.branch)"
```

**Task 2: `list-contents-task.yaml`**

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: list-contents-task
spec:
  params:
    - name: path
      description: The path to list contents of
      type: string
      default: "."
  workspaces:
    - name: source
      description: The workspace containing the directory to list
  steps:
    - name: list
      image: alpine
      script: |
        #!/bin/ash
        echo "Listing contents of $(workspaces.source.path)/$(params.path):"
        ls -la "$(workspaces.source.path)/$(params.path)"
```

Apply these two Tasks:

```bash
kubectl apply -f git-clone-task.yaml
kubectl apply -f list-contents-task.yaml
```

**Now, let's define the Pipeline: `simple-pipeline.yaml`**

This Pipeline will:
1.  Clone a Git repository using `git-clone-task`.
2.  List the contents of the cloned repository using `list-contents-task`.

Notice how the `workspaces` connect the output of the first Task to the input of the second.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: simple-repo-pipeline
spec:
  params:
    - name: gitRepoUrl
      description: The URL of the Git repository to clone
      type: string
    - name: gitBranch
      description: The Git branch to clone
      type: string
      default: main
  workspaces:
    - name: shared-workspace # This workspace will be shared between tasks
  tasks:
    - name: clone-repo
      taskRef:
        name: git-clone-task
      params:
        - name: repoUrl
          value: $(params.gitRepoUrl) # Pass pipeline parameter to task parameter
        - name: branch
          value: $(params.gitBranch)
      workspaces:
        - name: output
          workspace: shared-workspace # Connects task's 'output' workspace to pipeline's 'shared-workspace'

    - name: list-cloned-repo
      runAfter:
        - clone-repo # Ensures this task runs after 'clone-repo' completes
      taskRef:
        name: list-contents-task
      params:
        - name: path
          value: "." # List the root of the cloned repo
      workspaces:
        - name: source
          workspace: shared-workspace # Connects task's 'source' workspace to pipeline's 'shared-workspace'
```

Apply the Pipeline:

```bash
kubectl apply -f simple-pipeline.yaml
```

Verify the Pipeline is created:

```bash
kubectl get pipelines
```

---

### Step 5: Run the Pipeline using a PipelineRun

A **PipelineRun** is an instantiation of a Pipeline.

Create a file named `simple-pipelinerun.yaml`:

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: run-simple-repo-pipeline
spec:
  pipelineRef:
    name: simple-repo-pipeline # References the Pipeline we just created
  params:
    - name: gitRepoUrl
      value: https://github.com/tektoncd/triggers.git # Replace with any public Git repo URL
    - name: gitBranch
      value: main
  workspaces:
    - name: shared-workspace # Maps the pipeline's workspace to a persistent volume claim (PVC) or emptyDir
      volumeClaimTemplate: # For demonstration, using an ephemeral volume
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
```

Apply the PipelineRun:

```bash
kubectl apply -f simple-pipelinerun.yaml
```

Now, watch the logs of your PipelineRun:

```bash
tkn pipelinerun logs run-simple-repo-pipeline -f
```

You'll see logs from both tasks, starting with the `git-clone-task` and then the `list-contents-task` showing the directory contents of the cloned repository.

This demonstrates how Tekton breaks down CI/CD workflows into reusable Tasks, which can then be orchestrated into Pipelines, leveraging Kubernetes for execution and resource management.

---

### Cleanup (Optional)

To remove the resources created in this example:

```bash
kubectl delete -f simple-pipelinerun.yaml
kubectl delete -f simple-pipeline.yaml
kubectl delete -f git-clone-task.yaml
kubectl delete -f list-contents-task.yaml
kubectl delete -f echo-taskrun.yaml
kubectl delete -f echo-task.yaml
# To uninstall Tekton entirely:
# kubectl delete --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```

## Overview

A `Task` is a collection of `Steps` that you define and arrange in a specific order
of execution as part of your continuous integration flow. A `Task` executes as a
Pod on your Kubernetes cluster. A `Task` is available within a specific namespace,
while a cluster resolver can be used to access Tasks across the entire cluster.

A `Task` declaration includes the following elements:

- [Parameters](#specifying-parameters)
- [Steps](#steps)
- [Workspaces](#specifying-workspaces)
- [Results](#emitting-results)

Here's a basic Tekton `Task` and `TaskRun` YAML file to clone the "[https://github.com/mnakib/Podman-Labs.git](https://github.com/mnakib/Podman-Labs.git)" repository.


-----

**1. `clone-repo-task.yaml` (The Tekton Task)**

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: clone-repo-task
spec:
  params:
    - name: repo-url
      description: The URL of the Git repository to clone.
      type: string
      default: "https://github.com/mnakib/Podman-Labs.git"
    - name: target-path
      description: The path where the repository should be cloned.
      type: string
      default: "$(workspaces.output.path)/$(params.repo-name)"
    - name: repo-name
      description: The name of the repository (used for directory creation).
      type: string
      default: "Podman-Labs"
  workspaces:
    - name: output
      description: The workspace where the repository will be cloned.
  steps:
    - name: clone-repo
      image: alpine/git
      script: |
        #!/bin/ash
        echo "Cloning repository: $(params.repo-url) into $(params.target-path)"
        git clone $(params.repo-url) $(params.target-path)
        echo "Repository cloned. Listing contents:"
        cd $(params.target-path)
        ls -la
```

-----

**2. `clone-repo-taskrun.yaml` (The Tekton TaskRun)**

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: clone-podman-labs-repo-run
spec:
  taskRef:
    name: clone-repo-task
  params:
    - name: repo-url
      value: "https://github.com/mnakib/Podman-Labs.git"
    - name: target-path
      value: "/workspace/output/podman-labs-repo" # Example of overriding default path
    - name: repo-name
      value: "Podman-Labs"
  workspaces:
    - name: output
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
```

-----

**How to Use:**

1.  **Save the files:** Save the first YAML as `clone-repo-task.yaml` and the second as `clone-repo-taskrun.yaml`.
2.  **Apply the Task:**
    ```bash
    kubectl apply -f clone-repo-task.yaml
    ```
3.  **Apply the TaskRun:**
    ```bash
    kubectl apply -f clone-repo-taskrun.yaml
    ```
4.  **Monitor the TaskRun:**
    ```bash
    kubectl get taskruns -w
    ```
    You should see the `clone-podman-labs-repo-run` in action.
5.  **View logs (after completion or if it's running):**
    ```bash
    kubectl logs -f pod/$(kubectl get pod -l tekton.dev/taskRun=clone-podman-labs-repo-run -o jsonpath='{.items[0].metadata.name}') -c clone-repo
    ```
    This will show you the output of the `git clone` command and the `ls -la` command, confirming the repository was cloned.

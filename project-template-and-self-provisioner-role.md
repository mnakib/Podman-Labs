
# The Project Template and the Self-Provisioner Role

## Motivations

OpenShift introduces the `ProjectRequest` resource type. When you create a project request, the OpenShift API server creates a namespace from a template. By using a template, cluster administrators can customize namespace creation. For example, cluster administrators can ensure that new namespaces have specific permissions, resource quotas, or limit ranges.

Cluster administrators can allow users to create namespaces. Administrators can also customize the creation of namespaces to ensure that namespaces follow organizational requirements.

You can add any namespaced resource to the project template. For example, you can add resources of the following types:

- Roles and role bindings
- Resource quotas and limit ranges
- Network policies

## Creating a Project Template

The `oc adm create-bootstrap-project-template` command prints a template that you can use to create your own project template.

```sh
$ oc adm create-bootstrap-project-template -o yaml > file
```
This initial template has the following content:

```yaml
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  creationTimestamp: null
  name: project-request
objects: 
- apiVersion: project.openshift.io/v1 
  kind: Project
  metadata:
    annotations:
      openshift.io/description: ${PROJECT_DESCRIPTION}
      openshift.io/display-name: ${PROJECT_DISPLAYNAME}
      openshift.io/requester: ${PROJECT_REQUESTING_USER}
    creationTimestamp: null
    name: ${PROJECT_NAME}
  spec: {}
  status: {}
- apiVersion: rbac.authorization.k8s.io/v1 
  kind: RoleBinding
  metadata:
    creationTimestamp: null
    name: admin
    namespace: ${PROJECT_NAME}
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: admin
  subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: ${PROJECT_ADMIN_USER}
parameters: 
- name: PROJECT_NAME
- name: PROJECT_DISPLAYNAME
- name: PROJECT_DESCRIPTION
- name: PROJECT_ADMIN_USER
- name: PROJECT_REQUESTING_USER
```

Modify the object list to add the required resources for new namespaces.

The YAML output of oc commands that return lists of objects is formatted similarly to the template objects key.

```sh
$ oc get limitrange,resourcequota -o yaml
```
```yaml
apiVersion: v1
items:
- apiVersion: v1
  kind: LimitRange
  metadata:
    creationTimestamp: "2024-01-31T17:48:23Z"
    name: example
    namespace: example
    resourceVersion: "881771"
    uid: d0c19c60-00a9-4028-acc5-22680f1ea658
  spec:
    limits:
    - default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 250m
        memory: 256Mi
      max:
        cpu: "1"
        memory: 1Gi
      min:
        cpu: 125m
        memory: 128Mi
      type: Container
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    creationTimestamp: "2024-01-31T17:48:04Z"
    name: example
    namespace: example
    resourceVersion: "881648"
    uid: 108f0771-dc11-4289-ae76-6514d58bbece
  spec:
    hard:
      count/pods: "1"
  status:
...
kind: List
metadata:
  resourceVersion: ""
```


## Configuring the Project Template

Update the `projects.config.openshift.io/cluster` resource to use the new project template. Modify the spec section. By default, the name of the project template is `project-request`.



## Managing Self-provisioning Permissions

Users with the self-provisioner cluster role can create projects. By default, the self-provisioner role is bound to all authenticated users.

Control the binding of the role to limit which users can request new projects.

```sh
$ oc describe clusterrolebinding.rbac self-provisioners
Name:         self-provisioners
Labels:       <none>
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
Role:
  Kind:  ClusterRole
  Name:  self-provisioner
Subjects:
  Kind   Name                        Namespace
  ----   ----                        ---------
  Group  system:authenticated:oauth
```

This role binding has an `rbac.authorization.kubernetes.io/autoupdate` annotation. This annotation protects roles and bindings from modifications that can interfere with the working of clusters. When the API server starts, the cluster restores resources with this annotation automatically, unless you set the annotation to the false value.

To disable permanently _self-provisioning_, execute below commands:

```sh
$ oc annotate clusterrolebinding/self-provisioners \
  --overwrite rbac.authorization.kubernetes.io/autoupdate=false
```
```
clusterrolebinding.rbac.authorization.k8s.io/self-provisioners annotated
```
```sh
$ oc patch clusterrolebinding.rbac self-provisioners \
  -p '{"subjects": null}'
```
```sh
clusterrolebinding.rbac.authorization.k8s.io/self-provisioners patched
```

You can also use the oc edit command to modify any value of a resource. The command launches the vi editor to apply your modifications. For example, to change the subject of the role binding from the system:authenticated:oauth group to the provisioners group, execute the following command:

```sh
$ oc edit clusterrolebinding/self-provisioners
```
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
...
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: self-provisioner
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: provisioners
```


## Example of configuring the project template

This example will show you how to restrict the ability to self-provision projects to a group of users, and ensure that all users from that group have write privileges on all projects that any of them creates. Also, ensure that their new projects are constrained by a limit range that restricts memory usage.

```sh
$ oc create namespace template-test
```

```sh
$ cat limitrange.yaml
```

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: max-memory
  namespace: template-test
spec:
  limits:
  - max:
      memory: 1Gi
    type: Container
```

```sh
$ oc apply -f limitrange.yaml
```

```sh
$ oc new-app --name test-deploy httpd
```

The limit range maximum prevents the deployment from creating pods.

```sh
$ oc get pod -n template-test
```

```
$ oc get event -n template-test \
  --sort-by=metadata.creationTimestamp
```
```
LAST SEEN   TYPE      REASON              OBJECT                       MESSAGE
...output omitted...
39s         Warning   FailedCreate        replicaset/test-846769884c   Error creating: pods "deploy-test-846769884c-5zjhw" is forbidden: maximum memory usage per Container is 1Gi, but limit is 2Gi
```

### Define the project template.

Use the `oc adm create-bootstrap-project-template` command to print an initial project template. Redirect the output to the **template.yaml** file.

```sh
$ oc adm create-bootstrap-project-template \
  -o yaml > template.yaml
```

Use the `oc` command to list the limit range in YAML format. Redirect the output to append to the **template.yaml** file

```sh
$ oc get limitrange -n template-test -o yaml >> template.yaml
```

Edit the **template.yaml** file to perform the necessary changes.

```yaml
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  creationTimestamp: null
  name: project-request
objects:
- apiVersion: project.openshift.io/v1
  kind: Project
  metadata:
    annotations:
      openshift.io/description: ${PROJECT_DESCRIPTION}
      openshift.io/display-name: ${PROJECT_DISPLAYNAME}
      openshift.io/requester: ${PROJECT_REQUESTING_USER}
    creationTimestamp: null
    name: ${PROJECT_NAME}
  spec: {}
  status: {}
- apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    creationTimestamp: null
    name: admin
    namespace: ${PROJECT_NAME}
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: admin
  subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: Group
    name: provisioners
- apiVersion: v1
  kind: LimitRange
  metadata:
    name: max-memory
    namespace: ${PROJECT_NAME}
  spec:
    limits:
    - default:
        memory: 1Gi
      defaultRequest:
        memory: 1Gi
      max:
        memory: 1Gi
      type: Container
parameters:
- name: PROJECT_NAME
- name: PROJECT_DISPLAYNAME
- name: PROJECT_DESCRIPTION
- name: PROJECT_ADMIN_USER
- name: PROJECT_REQUESTING_USER
```

### Create and configure the project template.

Create the project template

```sh
$ oc create -f template.yaml -n openshift-config
```

Change the global cluster project configuration.

```sh
$ oc edit projects.config.openshift.io cluster
```

```
apiVersion: config.openshift.io/v1
kind: Project
metadata:
...
  name: cluster
...
spec:
  projectRequestTemplate:
    name: project-request
```

Check the API server pods

```sh
$ watch oc get pod -n openshift-apiserver
NAME		    READY   STATUS    RESTARTS	  AGE
apiserver-6b7b...   2/2	    Running   0           2m30s
```

### Create a project

```sh
$ oc login -u developer -p developer
```

```sh
$ oc new-project test
```
```
Now using project "test" on server "https://api.ocp4.example.com:6443".
```

Create a deployment that exceeds the limit range by using the **./deployment.yaml** file

Examine the pods and events in the _template-test_ namespace.

```sh
$ oc get pod
```
```sh
No resources found in test namespace.
```

The limit range works as expected.

```sh
$ oc get event --sort-by=metadata.creationTimestamp
```
```
LAST SEEN   TYPE      REASON              OBJECT                       MESSAGE
...output omitted...
39s         Warning   FailedCreate        replicaset/test-846769884c   Error creating: pods "test-846769884c-5zjhw" is forbidden: maximum memory usage per Container is 1Gi, but limit is 2Gi
```

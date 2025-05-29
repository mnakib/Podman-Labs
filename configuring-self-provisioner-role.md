# The Self-Provisioner Role

## Motivations

In OpenShift, the self-provisioner role grants users the ability to create new projects within the cluster. This role is typically assigned to all authenticated users by default. Essentially, it allows users to provision their own isolated spaces for deploying and managing applications.   

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

You can also use the oc edit command to modify any value of a resource. The command launches the vi editor to apply your modifications. For example, to change the subject of the role binding from the `system:authenticated:oauth` group to the `provisioners` group, execute the following command:

```sh
$ oc edit clusterrolebinding/self-provisioners
```
```yaml
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

> You need to have the `provisioners` group created in OpenShift and has already a few users as members.
> Follow the steps in (Managing users with the htpasswd identity provider)[https://github.com/mnakib/Podman-Labs/blob/main/managing-users-with-the-htpasswd-identity-provider.md] to configure the OpenShift cluster with local users from HTPasswd, then create a group named `provisioners` and add some users to it.
> 
> `$ oc adm groups new provisioners`
> 
> `$ oc adm groups add-users provisioners <user-name>`

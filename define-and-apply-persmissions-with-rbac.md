
# Define and Apply Permissions with RBAC

This example will guide you through defining role-based access controls and applying permissions to users.

- The tasks performed is this examples are:

- Remove project creation privileges from users who are not OpenShift cluster administrators.
- Create OpenShift groups and add members to these groups.
- Create a project and assign project administration privileges to the project.
- As a project administrator, assign read and write privileges to different groups of users.

### Check the `self-provisioner` cluster role assignment

Determine which cluster role bindings assign the `self-provisioner` cluster role

```sh
oc login -u admin -p redhatocp https://api.ocp4.example.com:6443
```

```sh
oc get clusterrolebinding -o wide | grep -E 'ROLE|self-provisioner'
```
```
NAME              ROLE                         ... GROUPS                     
self-provisioners ClusterRole/self-provisioner ... system:authenticated:oauth
```




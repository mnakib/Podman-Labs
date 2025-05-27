
# Managing Users with the HTPasswd Identity Provider

Managing user credentials with the HTPasswd Identity Provider requires creating a temporary htpasswd file, changing the file, and applying these changes to the secret.

Below are the listed steps:

- [Creating an HTPasswd File](https://github.com/mnakib/Podman-Labs/blob/main/managing-users-with-the-htpasswd-identity-provider.md#creating-an-htpasswd-file).
- [Creating the HTPasswd Secret](https://github.com/mnakib/Podman-Labs/blob/main/managing-users-with-the-htpasswd-identity-provider.md#creating-the-htpasswd-secret).
- [Configuring the HTPasswd Identity Provider](https://github.com/mnakib/Podman-Labs/blob/main/managing-users-with-the-htpasswd-identity-provider.md#configuring-the-htpasswd-identity-provider).

> The `apache2-utils` package needs to be installed to be able to use the htpasswd utility
> '''sh
> sudo apt install apache2-utils
> ```

### Creating an HTPasswd File

Create the .htpasswd file and add two users: admin and developer

```sh
htpasswd -c -B -b .htpasswd admin redhat
```

```sh
htpasswd -b .htpasswd developer developer
```

```sh
cat .htpasswd
```
```sh
admin:$2y$05$qQaFbpx4hbf4uZe.SMLSduTN8uN4DNJMJ4jE5zXDA57WrTRlpu2QS
developer:$apr1$S0TxtLXl$QSRfBIufYP39pKNsIg/nD1
```

### Creating the HTPasswd Secret

Log in to OpenShift and create a secret that contains the HTPasswd users file.

```sh
oc login -u admin -p redhatocp https://api.ocp4.example.com:6443
```

```sh
oc create secret generic localusers \
    --from-file htpasswd=.htpasswd -n openshift-config
```

Assign the admin user the cluster-admin role

```sh
oc adm policy add-cluster-role-to-user cluster-admin admin
```

###  Configuring the HTPasswd Identity Provider

Update the HTPasswd identity provider for the cluster so that your users can authenticate. Configure the custom resource file and update the cluster.

Export the existing OAuth resource to a file named oauth.yaml

```sh
oc get oauth cluster -o yaml > ./oauth.yaml
```

Edit the `oauth.yaml` file and choose the names of the `identityProviders` and `fileData` structures. For this example, use the `httpdusers` and `localusers` values, respectively.

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
...
spec:
  identityProviders:
  - ldap:  # **LDAP Identity Provider**
  ...
    type: LDAP
  - htpasswd:  # **HTPasswd Identity Provider**
      fileData:
        name: localusers
    mappingMethod: claim
    name: myusers
    type: HTPasswd
```

Apply the custom resource.

```bash
oc replace -f ./oauth.yaml
```

Authentication changes require redeploying pods in the openshift-authentication namespace.

Use the `watch` command to examine the status of workloads in the openshift-authentication namespace.

```sh
watch oc get all -n openshift-authentication
```
```
NAME                                   READY   STATUS    RESTARTS   AGE
pod/oauth-openshift-6d68ffb9dc-6f8dr   1/1     Running   3          2m
```

Log in as the new_admin and as the new_developer user to verify the HTPasswd user configuratio

```sh
oc login -u new_admin -p redhat
```

```sh
oc get nodes
```

```sh
oc get identity
```

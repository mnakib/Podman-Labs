# OpenShift Apache Server Rootless Image Configuration and Deployment

## The Dockerfile content

This Dockerfile contains the instructions to create a custom image exposing an httpd container on port 8080 

```bash
FROM docker.io/httpd

RUN sed -i 's/Listen 80/Listen 8080/g' /usr/local/apache2/conf/httpd.conf

RUN chgrp -R 0 /usr/local/apache2/logs && chmod -R g=u /usr/local/apache2/logs

COPY ./src/ /usr/local/apache2/htdocs/

# USER root

EXPOSE 8080

CMD ["httpd-foreground"]
```


## Create the image and push it Quay.io

```
# podman build -t apache-server .
```

```
# podman login quay.io -u <username> -p <password>
```

```
# podman push localhost/apache-server quay.io/mnakib/apache-server
```

## Deploy the image from OpenShift

Because this is a private image (unless you make it public), you will need to create a secret containing the private image registry credentials and link it to the account configured inside the deployment to be able to pull the image. The deployment is using the `default` account, so you'll just have to link to it. If you're using a different service account than `defautl`, the secret has to be linked to that custom service account.

Create the secret

```
# oc create secret docker-registry quay-creds \
--docker-username=<registry-username> \
--docker-password=<registry-password> \
--docker-server=quay.io
```

Link the secret to the service account

```
# oc secrets link default quay-creds --for=pull
```

Create the deployment

```
# oc create deployment apache-server --image=quay.io/mnakib/apache-server
```

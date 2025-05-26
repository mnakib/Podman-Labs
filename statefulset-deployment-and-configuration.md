# StatefulSet Configuration and Deployment

For deploying a containerized database in Kubernetes, it's generally better to use a StatefulSet rather than a Deployment for below reasons:

1 - Stable Identity (pod name persistence)

- StatefulSets provide persistent pod names (e.g., db-0, db-1, db-2).

- Pods in a StatefulSet start sequentially (one after another) and maintain order (db-0 starts before db-1).


2- Ordered Startup and Scaling

- Pods in a StatefulSet start sequentially (one after another) and maintain order (db-0 starts before db-1).

- Deployments start pods in parallel, which can lead to issues if your database requires an ordered startup (e.g., master-first replication setups).

3- Persistent Storage (PersistentVolumeClaim)

- StatefulSets ensure each pod gets its own dedicated persistent volume, which survives pod restarts.

- Deployments, unless carefully configured, often share storage, which can lead to data corruption in databases.

4- Pod-to-Pod Communication

- Each pod in a StatefulSet gets a stable network identity (db-0.db-service, db-1.db-service). This is essential for databases like MySQL, PostgreSQL, and Cassandra that need peer-to-peer communication.

- In a Deployment, pods get random names, which can complicate database clustering.

Below is are some examples of StatefulSet manifests for common databases in Kubernetes: MySQL, PostgreSQL, and MongoDB. These configurations ensure persistent storage, ordered scaling, and stable network identity.

### MySQL StatefulSet Example
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:9.3.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "rootpassword"
        - name: MYSQL_USER
          value: "user"
        - name: MYSQL_PASSWORD
          value: "pass"
        - name: MYSQL_DATABASE
          value: "testdb"
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```


### PostgreSQL StatefulSet Example
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
spec:
  serviceName: postgresql
  replicas: 2
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
      - name: postgresql
        image: postgres:latest
        env:
        - name: POSTGRES_USER
          value: "admin"
        - name: POSTGRES_PASSWORD
          value: "mypassword"
        - name: POSTGRES_DB
          value: "mydatabase"
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgresql-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgresql-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### MongoDB StatefulSet Example
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:latest
        args: ["--replSet", "rs0"]
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: mongodb-storage
          mountPath: /data/db
  volumeClaimTemplates:
  - metadata:
      name: mongodb-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```


## Expose the Stateful Set

**StatefulSets** in OpenShift **cannot be exposed using `oc expose`**, unlike Deployments or Services, because **StatefulSets manage pods differently**. Unlike Deployments, StatefulSets ensure **ordered, stable pod identities** and manage persistence. Another reason is that the **`oc expose` commands works with Services**. Typically, `oc expose` is used for `Service` resources, not StatefulSets. Hence, instead of exposing the StatefulSet directly, create a **Service** resource to expose.

It's worth noting as well that the service that need to be create is of type Headless. A Kubernetes Headless Service is used when you need direct access to the individual pods in a service, rather than routing traffic through a single virtual IP. It is maily used for stateful applications and in databases like MongoDB, Cassandra, or Elasticsearch, where each pod might have a unique identity and clients need to interact with specific instances. So instead of Kubernetes handling load balancing, applications can decide how to distribute requests among pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None # Enables direct pod communication
```

This creates a ClusterIP service for MySQL and exposes it.


## Create a WordPress Deployment

One way is to create a deployment from an S2I image and the WordPress git repo.

```sh
# oc new-app https://github.com/WordPress/WordPress.git
```

This command will create an image stream, a build config, a deployment, and a service.

...
--> Creating resources ...
    imagestream.image.openshift.io "wordpress" created
    buildconfig.build.openshift.io "wordpress" created
    deployment.apps "wordpress" created
    service "wordpress" created
--> Success
...

What is left to do is the creation of a route to access WordPress console from a browser.

```sh
# oc expose svc wordpress 
```

### Add a persistent volume to the WordPress deployment

```sh
# oc set volume deploy/wordpress --add --name=v1 -t pvc --claim-size=1G --mount-path /var/www/html/
```


---
title: Connecting API uptime monitoring dashboard and Integrating with Kubernetes
date: 2022-10-28
tags:
  - "cern analysis preservation"
  - blog
  - devops
  - monitoring
  - Parth Shandilya
  - "2022"
social_image: /media/rocket.jpg
description: This is a test description for SEO and Open Graph purposes. If
  it's not provided, it defaults to auto-generated excerpts of the page content.
---

## Why?

We put a lot of work and money into developing our APIs. We have tested it, tweaked it, and tested it some more. Everything works great during testing, and the live version does well.

Let’s face it, APIs fail for many reasons, so our investment in time needs to be protected. We owe it to the users of our API to ensure that it performs at optimum levels 24/7.

We need to monitor our APIs because,

- It is OUR responsibility.
- Regular website monitoring may not capture API issues.
- Not every API method may be working.
- Reduced user satisfaction

We can only be sure to conduct our own testing to protect our investment from third-party API failures.

## Monitoring Dashboard: Statping

We decided to integrate statping for API monitoring. 

Statping is an easy to use Status Page for websites and applications. Statping will automatically fetch the application and render a status page.


**Step 1: Adding the statping service in docker compose for development setup**

```yaml=
statping:
  image: statping/statping:v0.90.74
  restart: always
  ports:
    - "8080:8080"
  volumes:
    - ./docker/statping/app/config.yml:/app/config.yml
    - ./docker/statping/app/services.yml:/app/services.yml
  
```

**Step 2: Adding the custom config and services**

Now, we have mounted two files which will contain our config and services for statping setup. These files _should_ be present before the service startup in the mounted path.

we can provide the configuartion for setup of statping. Read in detail [here.](https://github.com/statping/statping/wiki/config.yml)

Example `config.yml`

```yaml=
connection: sqlite3
user: <username>
password: <password>
database: statping
api_secret: <secret>
language: en
allow_reports: true
location: /app 
sqlfile: /app/statping.db
disable_http: false
demo_mode: false
disable_logs: false
use_assets: false
sample_data: false
use_cdn: false
disable_colors: false
NAME: <Name of the Project>
DESCRIPTION: Monitoring Dashboard
```

We can import multiple services when Statping first loads by creating a file named services.yml in the working directory for Statping. It will insert the new service into the database, and will not be re-added on reboot.

Example: `service.yml`

```yaml=
services:
  - name: <Service Name>
    domain: {{host_url}}/ping
    expected: Pong!
    type: http
    method: GET
    headers: {{other_headers}},Authorization=Bearer {{token}}
    port: 80
    check_interval: 60
    timeout: 15
    expected_status: 200
    allow_notifications: true
    notify_after: 2
    notify_all_changes: true
    public: true
    redirect: true
```

We have a python script which populates the services.yml with the required `host_url`, `headers`.

Now we will move on to adding statping in our k8s cluster. We use _Kustomize_ which is a standalone tool to customize Kubernetes objects through a kustomization file.

**Step 3: Creating a Deployment for Statping in the cluster**

A Deployment provides declarative updates for Pods and ReplicaSets.

```yaml=
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: statping
spec:
  replicas: 1
  revisionHistoryLimit: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
  template:
    spec:
      volumes:
        - configMap:
            name: statping-cfg
          name: config
      initContainers: []
      containers:
        - name: statping
          image: hunterlong/statping:v0.90.74
          volumeMounts:
            - name: data
              mountPath: /app
            - name: config
              mountPath: /app/config.yml
              subPath: config.yml
          ports:
            - name: http
              containerPort: 8080
          envFrom:
            - secretRef:
                name: statping-creds
          env:
            - name: DOMAIN
                value: https://$(DOMAIN)/status
            - name: BASE_PATH
                value: status
```

**Step 4: Creating a Ingress for Statping in the cluster**

Ingress exposes HTTP and HTTPS routes from outside the cluster to services within the cluster. Traffic routing is controlled by rules defined on the Ingress resource.

```yaml=
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: statping
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-production
spec:
  tls:
    - hosts: [$(DOMAIN)]
      secretName: statping-cert
  rules:
    - host: $(DOMAIN)
      http:
        paths:
          - path: /status
            pathType: Prefix
            backend:
              service:
                name: statping
                port:
                  name: http
```

**Step 5: Creating a Service for Statping in the cluster**

Service is an abstract way to expose an application running on a set of Pods as a network service.

```yaml=
---
apiVersion: v1
kind: Service
metadata:
  name: statping
  annotations:
    prometheus.io/metrics_path: /status/metrics
spec:
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: http
    - name: metrics
      port: 9090
      protocol: TCP
      targetPort: http
```


**Step 6: Creating a ConfigMap and Secret for statping**

A ConfigMap is an API object used to store non-confidential data in key-value pairs. Pods can consume ConfigMaps as environment variables, command-line arguments, or as configuration files in a volume.

A ConfigMap allows you to decouple environment-specific configuration from your container images, so that your applications are easily portable.

ConfigMaps and Secrets hold configuration or sensitive data that are used by other Kubernetes objects, such as Pods. 

The source of truth of ConfigMaps or Secrets are usually external to a cluster.

Kustomize has secretGenerator and configMapGenerator, which generate Secret and ConfigMap from files or literals.

To use a ConfigMap in a Deployment, reference it by the name of the configMapGenerator. Kustomize will automatically replace this name with the generated name.

```yaml=

configMapGenerator:
  - name: statping-globals
    literals:
      - DOMAIN=<>
      - BACKUP_PATH=<>
  - name: statping-s3-globals
    literals:
      - FILE_NAME=<>
      - BUCKET_NAME=<>
      - S3_HOST=<>
      - STATPING_HOST_EXPORT_URL=<>
      - STATPING_HOST_IMPORT_URL=<>

```

We have statping configured in our cluster and ready to be deployed.

Now, we need to persist the data across pod restarts/failures. For acheiving that, we decided to go for a two way approach. 

We need to export the statping data on a regular interval that means implementing a cron job and import the data on pod restart that means implementing a side car for importing services just after the pod is up again.

**Step 7: Adding a Cronjob for exporting the statping data**

For the cronjob, we need a script that will export the services metadata with history and save it to S3 bucket. I created the script for export [here](https://github.com/cern-sis/statping-s3/blob/main/export-services.py). Now after the script, I need to run as a cronjob which will run every hour to export the data and keep it updated.

```yaml=
apiVersion: batch/v1
kind: CronJob
metadata:
  name: statping-export
spec:
  schedule: '@hourly'
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: statping-export
              image: registry.cern.ch/cern-sis/statping-s3:v1.18.0
              imagePullPolicy: IfNotPresent
              args: ['./export-services.py']
              envFrom:
                - secretRef:
                    name: statping-creds
                - secretRef:
                    name: statping-s3-creds
                - configMapRef:
                    name: statping-s3-globals
```


**Step 8: Adding a sidecar for importing the statping data on Statping Pod Restart**

This step was the challenging part and took most of the time since the import services is a script, and when the import is done, the Pod stops, both containers (Statping and the Import services sidecar) restart, and the container was reaching Backoff Limit. I created the script for import [here](https://github.com/cern-sis/statping-s3/blob/main/import-services.py).

The first way is to add `restartPolicy: OnFailure` to the container, but only `Always` is supported.

Another way was to keep the script running endlessly, and it worked 🎉

Below is the patch for adding sidecar container to the statping:

```yaml=
- name: statping-import
  image: registry.cern.ch/cern-sis/statping-s3:v1.18.0
  args: ['./import-services.py']
  envFrom:
    - secretRef:
        name: statping-creds
    - secretRef:
        name: statping-s3-creds
    - configMapRef:
        name: statping-s3-globals
```

We have now reached the end of this blog and Statping is ready to tested on qa and deployed on Production environments.

I personally learned a lot of things while intergrating the Statping dashboard. Thanks to Benjamin for helping me to understand Kubernetes concepts and actively answering to my questions.

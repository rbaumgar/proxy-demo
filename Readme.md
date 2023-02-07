# Reverse Proxy for Thanos-Querier in the OpenShift User Monitoring

When you want to use the Thanos-Querier in the OpenShift User Monitoring you have to make some additinial changes to get access to the prometheus data of your project.
- you have to present a authentication bearer to get access.
- you need to add "namespace=<namespace>" as query parameter. The bearer has to have at least view rights on the namespace.

Sometimes you are using a software where you are not able to define all required changes.
So I have created a simple nginx reverse proxy to help you.

## configuration

```shell
$ NAMESPACE=`oc project -q`
$ BEARER=`oc whoami -t`
$ cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx
data:
  nginx.conf: |
    # user  nginx;
    worker_processes  1;
    error_log  /var/log/nginx/error.log notice;
    pid        /var/run/nginx.pid;
    events {
        worker_connections  1024;
    }
    http {
      include       /etc/nginx/mime.types;
      default_type  application/octet-stream;
      log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';
      access_log  /var/log/nginx/access.log  main;
      sendfile        on;
      keepalive_timeout  65;
      server {
        listen 9092;

        rewrite_log on;

        location /healthz {
          return 200;
        }

        location / {
          rewrite ^(.*)$ $1?namespace=${NAMESPACE} break;

          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "Upgrade";
          proxy_pass https://thanos-querier.openshift-monitoring.svc.cluster.local:9092;
          proxy_set_header Host $host;
          proxy_set_header Authorization "Bearer ${BEARER}";
          proxy_http_version 1.1;
        }
      }
    }
EOF
configmap/nginx created
$ cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: registry.redhat.io/ubi8/nginx-120
          command: ["/bin/sh"]
          args: ["-c", "touch /var/log/nginx/access.log; nginx; tail -f /var/log/nginx/access.log"]
          ports:
          - containerPort: 9092
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf
      volumes:
        - name: nginx-config
          configMap:
            name: nginx
---
kind: Service
apiVersion: v1
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 9092
    targetPort: 9092
    name: nginx
EOF
deployment.apps/nginx created
service/nginx created
```

Check that the nxinx pod is up and runng

```shell
$ oc get pod -l app=nginx
NAME                     READY   STATUS    RESTARTS   AGE
nginx-76545df445-fj48l   1/1     Running   0          82m
```

Now you can check the access to your monitoring data from the thanos-querier via the nginx revers proxy.

```shell
$ oc rsh deployment/nginx curl "nginx.proxy-demo.svc:9092/api/v1/query?query=up" | jq
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "__name__": "up",
          "container": "otc-container",
          "endpoint": "metrics",
          "instance": "10.129.3.190:8889",
          "job": "my-otelcol-collector",
          "namespace": "jaeger-demo",
          "pod": "my-otelcol-collector-7d84758987-knv74",
          "prometheus": "openshift-user-workload-monitoring/user-workload",
          "service": "my-otelcol-collector"
        },
        "value": [
          1675784722.362,
          "1"
        ]
      }
    ]
  }
}
```

So if this is successful, you can now point your application to "http://nginx:9092" as the prometheus endpoint.

Have fun!
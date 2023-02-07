# Reverse Proxy for Thanos-Querier in the OpenShift User Workload Monitoring

When you want to use the Thanos-Querier in the OpenShift User Workload Monitoring you have to make some additinial changes to get access to the prometheus data of your project.
- you have to present a authentication bearer to get access.
- you need to add "namespace=<namespace>" as query parameter. The bearer has to have at least view rights on the namespace.

Sometimes you are using a software where you are not able to define all required changes.
So I have created a simple nginx reverse proxy to help you.

## configuration

```shell
$ NAMESPACE=`oc project -q`
$ BEARER=`oc whoami -t`
$ sed "s/<BEARER>/$BEARER/g" configmap.yaml | sed "s/<NAMESPACE>/$NAMESPACE/g" | oc apply -f -
configmap/nginx created
$ oc apply -f deployment.yaml
deployment.apps/nginx created
$ oc apply -f service.yaml
service/nginx created
```

Check that the nxinx pod is up and runng

```shell
$ oc get pod -l app=nginx
NAME                     READY   STATUS    RESTARTS   AGE
nginx-76545df445-fj48l   1/1     Running   0          82m
```

# test 

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
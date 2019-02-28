# OCP Application Monitoring

![Monitoring Component Diagram](diagram.png?raw=true "Monitoring Component Diagram")

## Deploy Prometheus Operator

`oc process -f operator/fuse-prometheus-operator.yml -o yaml -p NAMESPACE=<prometheus project> | oc apply -f -`

## Allow Prometheus to access other projects

### Give Prometheus permission to read metadata

Prometheus needs access to pod and service metadata in other projects via the Kubernetes API. A rolebinding template is provided.

Example command:

`oc process -f role-binding/rolebinding.yml -p PROMETHEUS_NAMESPACE=application-monitoring | oc apply -n dev -f -`

### Allow network connections to monitored namespaces

When your cluster is configured to use the ovs-multitenant SDN plug-in, you can manage the separate pod overlay networks for projects using the administrator CLI. So, if one Prometheus instance is intended to monitor any/all application projects, it may be suitable to make the Prometheus namespace global. Note that this involves some trade-offs from an environment isolation perspective and may contradict some best practices. *TODO: describe risks associated with this strategy*

`oc adm pod-network make-projects-global application-monitoring`

This will allow service monitors to target any namespace in the cluster.

Depending on requirements, it may be a better choice to explicitly join
the Prometheus namespace to the monitored namespace(s), rather than setting the project network to global on the Prometheus namespace.

`oc adm pod-network join-projects --to=application-monitoring dev`

`oc adm pod-network join-projects --to=application-monitoring test`

Note that you want to specify the Prometheus namespace as the `--to`. This way the Network ID of the Prometheus namespace remains
constant, and you will be able to add new projects to the network over time without listing all projects in the same join command. Note that this also involves some trade-offs from an environment isolation perspective and may contradict some best practices. *TODO: describe risks associated with this strategy*

To view which projects share the same Network ID, run:

`oc get netnamespaces`

If the numbers from the NETID column match, then those project networks are joined.

More info here: [OpenShift 3.11 - Managing Networking](https://docs.openshift.com/container-platform/3.11/admin_guide/managing_networking.html)

*Important:* Arguably, the ideal architecture involves a separate Prometheus instance for each monitored namespace, when the Multitenant is the cluster SDN implementation. The NetworkPolicy SDN implementation offers some more robust configuration options. Here's a [blog on NetworkPolicies and Microsegmentation](https://blog.openshift.com/networkpolicies-and-microsegmentation/) for an intro.



## Deploy ServiceMonitors

Create a Fuse service monitor:

`oc process -f service-monitors/fuse-servicemonitor.yml -p FUSE_SERVICE_NAME=my-service | oc apply -f -`

The target service must have a port exposed that matches the endpoint section of the ServiceMonitor

```
- apiVersion: v1
  kind: Service
  metadata:
    name: example-fuse
    labels:
      app: example-fuse
  spec:
    selector:
      app: example-fuse
    ports:
    - name: prometheus
      port: 9779
```

TODO: Revisit this. Is it necessary to add a service port for Prometheus? I'm not fully understanding what a service has to do with anything. We are targeting pods individually. Need to research.


## Deploy Grafana

`oc process -f grafana/deployment.yml -o yaml -p NAMESPACE=<prometheus project>  | oc apply -f -`

When you first connect your browser to Grafana, you'll need to provide a data source. Select "Prometheus" and use `http://prometheus:9090`.

You can import sample dahsboards from the dashboards directory in this repo.

## Deploy PrometheusRules

Choose one of the rules samples and apply in your Prometheus project, for example:

`oc apply -f rules/high-heap-usage.yml`

## Deploy AlertManager

Make sure you are in your Prometheus project.

The following secret contains all alert manager config including SMTP connection data, email templates and routing rules.

`oc create secret generic alertmanager-main --from-file=alert-manager/alertmanager.yaml`

`oc apply -f alert-manager/deployment.yml`

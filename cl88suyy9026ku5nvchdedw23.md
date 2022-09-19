## Civo Cloud k3s: Enable Traefik Dashboard

*This article is out of date.*

In this post I wanted to showcase how you can get the traefik dashboard enabled on the default civo cloud kubernetes k3s cluster. (It’s currently in beta, you can request access here). I must add, I’ve only played around with this for around an hour, I’m sure there are better ways (I don’t have that much experience with helm, I couldn’t edit the deployed version.)

The k3s cluster has been setup following [the civo kubernetes guide](https://www.civo.com/learn/kubernetes-cluster-administration-using-civo-cli).

The default helm chart has the dashboard disabled by default. The default options can be seen on [the helm chart github page](https://github.com/helm/charts/tree/master/stable/traefik#configuration). The `toml` file is stored as a configmap, we can just simply overwrite it:

```
apiVersion: v1
data:
  traefik.toml: |
    # traefik.toml
    logLevel = "info"
    defaultEntryPoints = ["http","https"]
    [entryPoints]
      [entryPoints.http]
      address = ":80"
      compress = true
      [entryPoints.https]
      address = ":443"
      compress = true
        [entryPoints.https.tls]
          [[entryPoints.https.tls.certificates]]
          CertFile = "/ssl/tls.crt"
          KeyFile = "/ssl/tls.key"
    [ping]
    entryPoint = "http"
    [kubernetes]
      [kubernetes.ingressEndpoint]
      publishedService = "kube-system/traefik"
    [traefikLog]
      format = "json"
    [api]
      dashboard = true
kind: ConfigMap
metadata:
  labels:
    app: traefik
    chart: traefik-1.76.1
    heritage: Tiller
    release: traefik
  name: traefik
  namespace: kube-system
```

Take a look at [the traefik api definition page](https://docs.traefik.io/v1.7/configuration/api/) and make sure you add the missing [api] entries to the file. Once you save the file we need to apply the configuration and need to restart the traefik pod so the toml configuration file is reload.

```
kubectl apply -f traefik-configmap.yaml
kubectl get pods -n kube-system
# ...
# traefik-6f7cd584cb-l9nsn         1/1     Running     0          10m
kubectl delete pod traefik-6f7cd584cb-l9nsn
```

Once you tail the logs of the pod you should see that the new pod has re-read the configuration file and has also started on port 8080.

```
{"level":"info","msg":"Server configuration reloaded on :8080","time":"2019-10-02T19:13:40Z"}
```

At this point the container is listening on port 8080, we need to modify the deployment of traefik so the admin port is exposed

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
  generation: 2
  labels:
    app: traefik
    chart: traefik-1.76.1
    heritage: Tiller
    release: traefik
  name: traefik
  namespace: kube-system
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: traefik
      release: traefik
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: traefik
        chart: traefik-1.76.1
        heritage: Tiller
        release: traefik
    spec:
      containers:
      - args:
        - --configfile=/config/traefik.toml
        image: traefik:1.7.12
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /ping
            port: 80
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 2
        name: traefik
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        - containerPort: 8880
          name: httpn
          protocol: TCP
        - containerPort: 443
          name: https
          protocol: TCP
        - containerPort: 8080
          name: admin
          protocol: TCP
        readinessProbe:
          failureThreshold: 1
          httpGet:
            path: /ping
            port: 80
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 2
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /config
          name: config
        - mountPath: /ssl
          name: ssl
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: traefik
      serviceAccountName: traefik
      terminationGracePeriodSeconds: 60
      volumes:
      - configMap:
          defaultMode: 420
          name: traefik
        name: config
      - name: ssl
        secret:
          defaultMode: 420
          secretName: traefik-default-cert
```

Save the file and let’s update the deployment:

```
kubectl apply -f traefik-deployment.yaml
```

To secure the dashboard we will just use basic auth, it’s simple and it works. I don’t want to repeat documentation already available, so please, see [the traefik documentation](https://docs.traefik.io/v1.7/user-guide/kubernetes/#basic-authentication) on how to create the credentials.

The last remaining step is to create a service for the dashboard and actually expose it via traefik:

```
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  selector:
    app: traefik
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      name: admin
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
    ingress.kubernetes.io/auth-type: "basic"
    ingress.kubernetes.io/auth-secret: "mysecret"
spec:
  rules:
  - host: <your-domain.fqdn>
    http:
      paths:
      - backend:
          serviceName: traefik-web-ui
          servicePort: 80
```

Let’s apply it:

```
kubectl apply -f traefik-admin.yaml
```

The last step is to actually point your domain to your Civo Kubernetes endpoint IP. The IP address can be found via the civo CLI tool:

```
civo kubernetes show <name of the cluster> | grep API
```

If you want to use Traefik v2, please check out [this blog post](https://akos.me/2019/civo-cloud-treafik-v2-tls/).

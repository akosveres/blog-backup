## Civo Cloud k3s: Traefik v2 and Let's Encrypt

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1663594001466/9SWnSOIz1.png align="center")

I’ve continued to play around with the K3S Beta cluster from Civo Cloud (you can request to join the beta), the aim was to also try out the new [Traefik V2](https://blog.containo.us/traefik-2-0-6531ec5196c2) which was released some time ago.

In this post we will:
- Update the default traefik install on k3s to v2.0.1
-     Secure the k3s install
-     Create and secure the ingress to the traefik dashboard
-     Enable Let's Encrypt certs for traefik
-     Some thoughts after the journey

Let’s start.
## Update Traefik to V2

By default the k3s cluster comes with Traefik v1.7. It’s a sane default, nothing wrong with it. As I am using a k3s cluster on a beta service, seemed like the perfect opportunity to play around with Traefik v2, it gives me the opportunity of exploring the new configuration, getting used to serving k8s services, as it matures, I’ll be ready to switch my production cluster to it.

The documentation around v2 is missing a lot of information, I’ve had to jump around quite a lot to understand how to get it going. (The Traefik documentation is open source, I could definetely contribute to it and it has been on my mind.) Through the post I’ll link to the relevant pages.
### Delete the existing traefik deployment and service

The existing Traefik deployment and service are under the name traefik, we just need to delete them:

```
kubectl delete deployment traefik -n kube-system
kubectl delete service traefik -n kube-system
```

### Deploy Traefik v2

There is [an official user guide](https://docs.traefik.io/user-guides/crd-acme/) that helps you setup Traefik with Let’s Encrypt enabled on minikube, 75% of it is totally usable in our setup as well. We won’t enable Let’s Encrypt from the beginning, we will need to take some extra steps to do so.

Let’s create the CRD definitions, I’m using the kube-system namespace, not the default one as it is in the guide:

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutes.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRoute
    plural: ingressroutes
    singular: ingressroute
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutetcps.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRouteTCP
    plural: ingressroutetcps
    singular: ingressroutetcp
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: middlewares.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: Middleware
    plural: middlewares
    singular: middleware
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: tlsoptions.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TLSOption
    plural: tlsoptions
    singular: tlsoption
  scope: Namespaced

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller

rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - traefik.containo.us
    resources:
      - middlewares
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - traefik.containo.us
    resources:
      - ingressroutes
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - traefik.containo.us
    resources:
      - ingressroutetcps
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - traefik.containo.us
    resources:
      - tlsoptions
    verbs:
      - get
      - list
      - watch

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
  - kind: ServiceAccount
    name: traefik-ingress-controller
    namespace: kube-system
```

Now we’re ready to create the deployment file together with the `ServiceAccount` used by Traefik:

```
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: kube-system
  name: traefik-ingress-controller

---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  namespace: kube-system
  name: traefik
  labels:
    app: traefik

spec:
  replicas: 1
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      labels:
        app: traefik
    spec:
      serviceAccountName: traefik-ingress-controller
      containers:
        - name: traefik
          image: traefik:v2.0
          args:
            - --log.level=INFO
            - --api.dashboard=true
            - --api.insecure=true
            - --accesslog
            - --global.sendanonymoususage
            - --entrypoints.web.Address=:80
            - --entrypoints.websecure.Address=:443
            - --providers.kubernetescrd
          ports:
            - name: web
              containerPort: 80
            - name: websecure
              containerPort: 443
            - name: admin
              containerPort: 8080
```

Explanation on some of the options (args) passed, the full list of options can be find on [the CLI Reference Page](https://docs.traefik.io/reference/static-configuration/cli/):
-      `--log.level=INFO` - By tailing the logs, we will be able to see more information
-     `--api.dashboard` - Enable the Traefik Dashboard
-     `--api.insecure` - Enable access to the Dashboard, we will secure this with basic auth later on
-     `--acceslog` - Enable access logs
-     `--global.sendanonymoususage` - It’s a test setup, the guys from Traefik maybe can find some anonymous data useful. Details of the collected data can be found [here](https://docs.traefik.io/contributing/data-collection/).

Last piece of the puzzle is the service:

```
apiVersion: v1
kind: Service
metadata:
  name: traefik
  namespace: kube-system
spec:
  ports:
    - protocol: TCP
      name: http
      port: 80
      nodePort:
    - protocol: TCP
      name: admin
      port: 8080
    - protocol: TCP
      name: https
      port: 443
      nodePort:
  selector:
    app: traefik
  type: LoadBalancer
```

Let’s deploy what we have right now:

```
kubectl apply -f traefik-crd.yaml
kubectl apply -f traefik-deployment.yaml
kubectl apply -f traefik-svc.yaml
```

In a couple of seconds the pod should start and the service should be able to serve content. You can check all of these with some of the following commands:

```
kubectl get pods -n kube-system
kubectl logs traefik-<random-hash> -n kube-system
kubectl get svc -n kube-system
```

## Secure the Civo Cloud k3s install

Civo Cloud has almost all ports open by default, this has been discussed in the #kube100 Slack channel, it will be looked at and secured fully, until then we just have to do another manual approach. Each k3s cluster deployed has a firewall created for it, open your Civo Cloud dashboard, click on `Kubernetes`, click on the name of your cluster and you’ll see `Firewall: Kubernetes cluster: <name of your cluster>`, click on it and you’ll be presented with the page where you can modify the rules.

Keep in mind this is a test cluster, you shouldn’t have any valuable information stored inside of it, if you do, it’s totally up to you to keep it secured.

These are the ports I’ve kept open:

    ICMP N/A 0.0.0.0/0 Ping/Traceroute - Just so I’m able to ping the k3s nodes
    TCP 443 0.0.0.0/0 HTTPS - Access to the https endpoint for Traefik
    TCP 80 0.0.0.0/0 HTTP - Access to the http endpoint for Traefik, Let’s encrypt needs access to this so it can do the HTTP challenge
    TCP 6443 0.0.0.0/0 K3S API - Access to the secure k3s API

If you want to secure the cluster even more, feel free to use your own public IP instead of 0.0.0.0/0, if the IP is dynamic, don’t forget to update it.

## Create and secure the access to the Traefik Dashboard

The k3s nodes all have public IP addresses, Traefik runs the dashboard on port 8080, with the service created, by default it means anyone who opens one of the public IPs on port 8080 can have unauthenticated access to the dashboard. We mitigated this risk by not allowing port 8080 to be open to these public IPs. In order to go around it we will create an IngressRule on a specific host that will route all of our requests to the Traefik Dashboard.

### Point domain to Public IP

The first step is to make sure you point a domain to one of the public IP addresses. I’ve chosen the same public IP where the kubectl context is configured, you can see this with the following command:

```
civo kubernetes show <name of your cluster> | grep API
```

Simply create an A record that points to the public IP Address. We also need this step for the Let’s Encrypt step.

### Create the IngressRule

We will set up an `IngressRule` that will route any requests coming in on our domain name to the Traefik dashboard, for security we will add basic auth.

The [basic auth documentation](https://docs.traefik.io/middlewares/basicauth/) page explain how it works in the new Traefik v2, the step I’ve missed was on how to craete the basic auth. With a bit of exploring, it turns out it’s the same as in the old version, you create the htpassd file, create a secret from it and you present it to Traefik:

```
htpasswd -c ./users <your username>
kubectl create secret generic mysecret --from-file users --namespace=kube-system
```

Time to create the IngressRoute and Middleware:

```
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: test-auth
  namespace: kube-system
spec:
  basicAuth:
    secret: mysecret
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  namespace: kube-system
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`<your-domain.fqdn>`)
    kind: Rule
    services:
    - name: traefik
      port: 8080
    middlewares:
      - name: test-auth
```

Let’s apply it:

```
kubectl apply -f traefik-ingress-dashboard.yaml
```

There we go, if you visit `http://your-domain.fqdn` you should be presented with basic auth, use your username and password and voila, the new great Traefik Dashboard will appear! Feel free to get used to the new layout and all the new information presented (I do miss the statistics, if I’m honest).

### Enable Let’s Encrypt

Traefik’s one big and great feature is that it can handle SSL Certs for you automagically, let’s use it.

Configuration to Traefik is passed via the `args` field in the configuration file, we modify the previously created file and we’ll make sure it looks like this:

```
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: kube-system
  name: traefik-ingress-controller

---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  namespace: kube-system
  name: traefik
  labels:
    app: traefik

spec:
  replicas: 1
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      labels:
        app: traefik
    spec:
      serviceAccountName: traefik-ingress-controller
      containers:
        - name: traefik
          image: traefik:v2.0
          args:
            - --log.level=INFO
            - --api.dashboard=true
            - --api.insecure=true
            - --accesslog
            - --global.sendanonymoususage
            - --entrypoints.web.Address=:80
            - --entrypoints.websecure.Address=:443
            - --providers.kubernetescrd
            - --certificatesresolvers.default.acme.tlschallenge
            - --certificatesresolvers.default.acme.email=<your email address@fqdn.com>
          ports:
            - name: web
              containerPort: 80
            - name: websecure
              containerPort: 443
            - name: admin
              containerPort: 8080
```

The two new fields:

> --certificatesresolvers.default.acme.tlschallenge - Enable TLS Challenge
>     --certificatesresolvers.default.acme.email=<your email address@fqdn.com - The email address used to register the SSL Certs, used only for communication

Save the file and apply it:

```
kubectl apply -f traefik-deployment.yaml
```

Traefik will restart the pods, it’s fast but until then we can simply update our IngressRoute to make use of the SSL enabled. Open the previously saved traefik-ingress-dashboard.yaml and update it:

```
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: test-auth
  namespace: kube-system
spec:
  basicAuth:
    secret: mysecret
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  namespace: kube-system
spec:
  entryPoints:
    - web
    - websecure
  routes:
  - match: Host(`<your-domain.fqdn>`)
    kind: Rule
    services:
    - name: traefik
      port: 8080
    middlewares:
      - name: test-auth
```

The difference between the two IngressRoutes are:
- `entryPoints` - Traefik has 3 entrypoints, http, https and admin. For https, port 443, we used the name `websecure`, you can see this in the deployment definition.
-     `tls.certResolver` - You can have multiple certResolvers, which is great, imagine using some protected, company owned cert provider but you also want to have public endpoints. The name of the certResolver is default, as defined in the `--certificatesresolvers.default.acme.tlschallenge` option in the deployments.

Let’s apply:

```
kubectl apply -f traefik-ingress-deployment.yaml
```

Once you go to `https://your-domain.fqdn` you should be presented with the basic auth, once you log in, the dashboard should present its self behind SSL in all of its glory! Congratulations, we have achieved something great today!
## Some thoughts after the journey
Civo Cloud

The beta k3s cluster presented by Civo Cloud had no issues, everything worked correctly. The configuration around it is not optimal at this point, but it’s a beta feature, we’re helping by testing the cluster its self. Network security around it will be addressed, at least that’s what I’ve seen from the discussions in slack. I’d like to see an option where I can simply disable the public IPs and just have a load balancer in front of it. Locked down security groups.

I want to work on some sort of automation on configuring the firewalls once they get created, I’ve seen some write up about using terraform with Civo Cloud, which is great news!

Civo Cloud does have the option for Load Balancers, I just didn’t look in to it long enough on how can I use one with the k3s cluster.
### Traefik

I am very excited about Traefik v2 for the simple fact that I’ve been using their product for two years now, I’ve seen it mature, it works in production, it’s just getting better. Especilly with the new v2 feature of [Observability](https://docs.traefik.io/observability/tracing/overview/), which I do want to look in to more, maybe that will be another post down the line. Then there is [Maesh](https://mae.sh/), the simple service mesh!

Documentation for v2 is scarce, though as mentioned, I could be contributing, so it’s on me as well. I’m very glad that I got v2 working on a k3s install, from here on it’s just me playing around with all the new options and I’m really curious to see what metrics are exposed for Prometheus, which is the next big thing I want to look at, especially that there are no stats in the new Traefik v2 Dashboard.
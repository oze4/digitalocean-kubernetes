<h1 align="center">How to Install Prometheus</h1>

- #### Please note, this setup assumes you are using an NGINX Ingress Controller!! 
- #### This setup also uses a `StatefulSet` to persist Prometheus data
- #### Change the `volumeClaimTemplates.spec.storageClassName` to fit your needs
- #### We also have [`cert-manager`](https://github.com/jetstack/cert-manager) configured within the `prometheus-ingress.yaml` file - you can just comment that out if you do not use `cert-manager`

<h2 align="center">*** Important Notes: ***</h2>

We expose Prometheus externally via and `Ingress` Resource. Prometheus does not have any feature which allows you to configure authentication.  If you would like to secure your exposed Prometheus instance, you can follow [this article](https://docs.nginx.com/nginx/admin-guide/security-controls/configuring-http-basic-authentication/#prerequisites) which outlines how to secure a service via `Basic Auth`

[If you want to expose Prometheus via NodePorts, follow this article](https://kubernetes.github.io/ingress-nginx/user-guide/monitoring/), although you shouldn't need to expose Prometheus externally for this to work.. It is worth noting that the paths supplied in that article are incorrect - use this path instead `github.com/kubernetes/ingress-nginx/tree/master/deploy/prometheus`

---

<h1 align="center">How-To:</h1>

- Download all of the ***YAML*** files that are in this directory to a folder on your computer
- Run: `kubectl apply --kustomize directory/you/saved/these/files`
- [Follow this guide](https://github.com/oze4/digitalocean-kubernetes/blob/master/grafana) to configure Grafana for Prometheus

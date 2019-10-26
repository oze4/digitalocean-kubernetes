# How to Install Prometheus

- ### Please note, this setup assumes you are using an NGINX Ingress Controller!! 
- ### We also have [`cert-manager`](https://github.com/jetstack/cert-manager) configured within the `prometheus-ingress.yaml` file - you can just comment that out if you do not use `cert-manager`

We also have cert-manager configured within the grafana-ingress.yaml file - you can just comment that out if you do not use cert-manager

[If you want to expose Prometheus via NodePorts, follow this article](https://kubernetes.github.io/ingress-nginx/user-guide/monitoring/), although you shouldn't need to expose Prometheus externally for this to work.. It is worth noting that the paths supplied in that article are incorrect - use this path instead `github.com/kubernetes/ingress-nginx/tree/master/deploy/prometheus`

---

# How-To:

- Download all of the ***YAML*** files that are in this directory to a folder on your computer
- Run: `kubectl apply --kustomize directory/you/saved/these/files`
- [Follow this guide](https://github.com/oze4/digitalocean-kubernetes/blob/master/grafana) to configure Grafana for Prometheus

# How to Install Prometheus

### Please note, this setup assumes you are using an NGINX Ingress Controller!! [If you want to expose Grafana via NodePorts, follow this article](https://kubernetes.github.io/ingress-nginx/user-guide/monitoring/)

<small>It is worth noting that the paths supplied in the article above are incorrect - use this path instead github.com/kubernetes/ingress-nginx/tree/master/deploy</small>

### We expose Grafana via an Ingress Resource

---

# How-To:

- Download all of the ***YAML*** files that are in this directory to a folder on your computer
- Run: `kubectl apply --kustomize directory/you/saved/these/files`
- #### Go to the `grafana` directory at the root of this repo to configure dashboards for Prometheus

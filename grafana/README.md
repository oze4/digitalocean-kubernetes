# How to Install Grafana

### Please note, this setup assumes you are using an NGINX Ingress Controller!! 
#### [If you want to expose Prometheus via NodePorts, follow this article](https://kubernetes.github.io/ingress-nginx/user-guide/monitoring/), although you shouldn't need to expose Prometheus at all

<small>It is worth noting that the paths supplied in the article above are incorrect - use this path instead github.com/kubernetes/ingress-nginx/tree/master/deploy</small>

---

# How-To:

- ### First, make sure you have Prometheus set up!
- Download all of the ***YAML*** files that are in this directory to a folder on your computer
- You do ***NOT*** need the `dashboard` directory for installation
- Run: `kubectl apply --kustomize directory/you/saved/these/files`
- Import the dashboard located in `/dashboard/dashboard.json`

## Great success

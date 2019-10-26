# How to Install Grafana

### Please note, this setup assumes you are using an NGINX Ingress Controller!! 

[If you want to expose Grafana via NodePorts, follow this article](https://kubernetes.github.io/ingress-nginx/user-guide/monitoring/).. It is worth noting that the paths supplied in that article are incorrect - use this path instead `github.com/kubernetes/ingress-nginx/tree/master/deploy/grafana`

---

# How-To:

- Make sure you have Prometheus set up!
- If you don't have Prometheus set up, [visit this Prometheus setup guide](https://github.com/oze4/digitalocean-kubernetes/tree/master/prometheus)
- Download all of the ***YAML*** files that are in this directory to a folder on your computer
- You do ***NOT*** need the `dashboard` directory for installation
- Run: `kubectl apply --kustomize directory/you/saved/these/files`
- Login to Grafana with username/password: `admin/admin`
- Once logged into Grafana, import the dashboard located in `/dashboard/dashboard.json`

## Great success

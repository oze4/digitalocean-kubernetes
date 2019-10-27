<h1 align="center">How to Install Grafana</h1>

- #### Please note, this setup assumes you are using an NGINX Ingress Controller!!
- #### This setup also uses a `StatefulSet` to persist Grafana data
- #### Change the <br/>`[grafana-stateful-set.yaml]spec.volumeClaimTemplates.spec.storageClassName` <br/>to fit your needs
- #### We also have [`cert-manager`](https://github.com/jetstack/cert-manager) configured within the `grafana-ingress.yaml` file - you can just comment that out if you do not use `cert-manager`

<h2 align="center">*** Important Notes: ***</h2>

[If you want to expose Grafana via NodePorts, follow this article](https://kubernetes.github.io/ingress-nginx/user-guide/monitoring/).. It is worth noting that the paths supplied in that article are incorrect - use this path instead `github.com/kubernetes/ingress-nginx/tree/master/deploy/grafana`

---

<h1 align="center">How-To:</h1>

- Make sure you have Prometheus set up!
- If you don't have Prometheus set up, [visit this Prometheus setup guide](https://github.com/oze4/digitalocean-kubernetes/tree/master/prometheus)
- Download all of the ***YAML*** files that are in this directory to a folder on your computer
- You do ***NOT*** need the `dashboard` directory for installation
- Run: `kubectl apply --kustomize directory/you/saved/these/files`
- Login to Grafana with username/password: `admin/admin`
- Once logged into Grafana, import the dashboard located in `/dashboard/dashboard.json`

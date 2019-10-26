# How to Install Prometheus

### Please note, this setup assumes you are using an NGINX Ingress Controller!!

### We expose Grafana via an Ingress Resource

- Download all of the ***YAML*** files that are in this directory to a folder on your computer
- Run: `kubectl apply --kustomize directory/you/saved/these/files`
- ## Go to the `grafana` directory at the root of this repo to configure dashboards for Prometheus

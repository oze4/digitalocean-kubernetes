# How to Install Grafana

### Please note, this setup assumes you are using an NGINX Ingress Controller!!

- ### First, make sure you have Prometheus set up!
- Download all of the ***YAML*** files that are in this directory to a folder on your computer
- You do ***NOT*** need the `dashboard` directory for installation
- Run: `kubectl apply --kustomize directory/you/saved/these/files`
- Import the dashboard located in `/dashboard/dashboard.json`

## Great success

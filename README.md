# Phonebook Microservice on Kubernetes (Flask + MySQL + Helm)

A phonebook web application split into two Flask microservices backed by MySQL,
deployed on a Kubernetes cluster (1 master + 1 worker on AWS EC2) and packaged as a Helm chart.

| Component | Role | Exposed on |
|---|---|---|
| web-server (`irfcan/webserver`) | Add / update / delete records | NodePort 30001 |
| result-server (`irfcan/resultserver`) | Search records (read-only) | NodePort 30002 |
| mysql (`mysql:5.7`) | Database, data kept on a PV/PVC (20Gi, hostPath `/mnt/data`) | ClusterIP 3306 (internal) |
User -> :30001 -> webserver-svc -> web pods --
User -> :30002 -> resultserver-svc -> result pods ---> mysql (ClusterIP) -> PVC -> PV

Optional: with Traefik Ingress everything is reachable on one port
(`/` -> web server, `/result` -> result server).

## Repository layout
image_for_web_server/ Flask app + Dockerfile (add/update/delete)
image_for_result_server/ Flask app + Dockerfile (search)
phonebook-chart/ Helm chart (Deployments, Services, ConfigMaps, Secret, PV/PVC, optional Ingress)

## Build and push the images

```bash
docker build -t <user>/webserver:latest image_for_web_server
docker build -t <user>/resultserver:latest image_for_result_server
docker push <user>/webserver:latest
docker push <user>/resultserver:latest
```

## Deploy

```bash
git clone https://github.com/irfcan/kubernetes-microservice-phonebook.git
cd kubernetes-microservice-phonebook
helm install phonebook phonebook-chart
kubectl get pods,svc,pv,pvc
```

Open `http://<NODE-PUBLIC-IP>:30001` (manage records) and `http://<NODE-PUBLIC-IP>:30002` (search).
The security group must allow ports 30001 and 30002.

Use your own images:
```bash
helm upgrade phonebook phonebook-chart \
  --set webserver_image=<user>/webserver:latest \
  --set resultserver_image=<user>/resultserver:latest
```

## Optional: Ingress with Traefik

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install traefik traefik/traefik -n traefik --create-namespace \
  --set service.type=NodePort --set ports.web.nodePort=30080
helm upgrade phonebook phonebook-chart --set ingress.enabled=true
```
Then open `http://<NODE-PUBLIC-IP>:30080/` and `http://<NODE-PUBLIC-IP>:30080/result`
(allow port 30080 in the security group).

## Persistence test

```bash
kubectl delete pod -l app=mysql-deploy
```
Once the new MySQL pod is running, the records added earlier are still returned by the search page.

## Notes

- The apps connect to MySQL once at startup. If MySQL was restarted, run
  `kubectl rollout restart deploy webserver-deploy resultserver-deploy`.
- The web server creates the `phonebook` table on startup, so it must have started at least once before searching.
- `hostPath` storage lives on a single node. The MySQL pod must run on the same node after a restart.
- The credentials in `phonebook-chart/templates/mysql-secret.yaml` are demo values. Do not reuse them anywhere real.

## Clean up

```bash
helm uninstall phonebook
```

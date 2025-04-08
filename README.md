# Ballerina Observability Starter Kit

Ovaj starter kit sadrži osnovnu Ballerina aplikaciju koja je integrisana sa Prometheus-om i Grafana-om za praćenje metrika. Takođe uključuje Helm chart za jednostavno postavljanje na Kubernetes klaster.

## Komponente

- **Ballerina aplikacija**: REST servis sa health endpointom i custom metrikama.
- **Prometheus**: Za prikupljanje metrika.
- **Grafana**: Za vizualizaciju metrika.
- **AlertManager**: Za slanje notifikacija na osnovu pravila.
- **Helm Chart**: Za jednostavan deploy na Kubernetes.

## Pre-rekviziti

Pre nego što počneš sa deploy-om, uveri se da imaš sledeće instalirano:
- Kubernetes klaster (lokalno ili cloud-based)
- Helm 3
- Prometheus i Grafana instalirani u klasteru
- Docker (ako želiš graditi slike lokalno)

## Deploy

### 1. Build Docker sliku za Ballerina aplikaciju

Prvo trebaš da izgradimo Docker sliku za tvoju Ballerina aplikaciju. U root direktorijumu repozitorijuma kreiraj `Dockerfile`:

```Dockerfile
FROM ballerina/ballerina:slbeta

COPY ballerina-app/ /home/ballerina/ballerina-app/

WORKDIR /home/ballerina/ballerina-app/

RUN ballerina build main.bal

CMD ["bal", "run", "main.bal"]
```

Zatim, izgradite Docker sliku:

```bash
docker build -t ballerina-user-service:latest .
```

### 2. Deploy sa Helm-om

Pre nego što implementiraš aplikaciju, trebaš da napraviš Prometheus monitore putem Helm-a.

1. Dodaj Prometheus repo (ako već nije dodan):

    ```bash
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update
    ```

2. Deploy Prometheus i Grafana:

    ```bash
    helm install prometheus prometheus-community/kube-prometheus-stack
    ```

3. Deploy tvoje Ballerina aplikacije sa Helm chart-om:

    ```bash
    helm install ballerina-app ./helm/ballerina-app
    ```

### 3. Pristup Prometheus i Grafana

- **Prometheus**: Nađeš ga putem port-forwarding-a:

    ```bash
    kubectl port-forward svc/prometheus-operated 9090:9090
    ```

    Onda, otvori [http://localhost:9090](http://localhost:9090) u tvom browseru.

- **Grafana**: Možeš pristupiti putem port-forwarding-a:

    ```bash
    kubectl port-forward svc/grafana 3000:80
    ```

    Zatim otvori [http://localhost:3000](http://localhost:3000) u tvom browseru. Korisničko ime i lozinka su `admin/admin`.

### 4. Upozorenja i Alerting

Prometheus će automatski početi pratiti tvoje metrike. Ako latencija pređe prag ili se aplikacija ugasi, AlertManager će poslati obaveštenje prema podešenim pravilima.

Da bi podesio e-mail notifikacije, izmeni `monitoring/alertmanager-config.yaml` fajl sa svojim SMTP podacima.

### 5. Health Endpoint

Aplikacija takođe izlaže health endpoint na portu 8081. Možeš proveriti status aplikacije putem:

```bash
curl http://<your-k8s-service-ip>:8081/healthz
```

Ako sve funkcioniše, trebalo bi da vidiš odgovor: `"OK"`.

## Dodatna podešavanja

Ako želiš da dodas dodatne metrike ili modifikujete postojeće, slobodno ažuriraj Ballerina kod u `ballerina-app/main.bal` i Helm chart konfiguraciju.

## Kontakt

Ako imaš bilo kakvih pitanja ili trebaš pomoć, slobodno se obrati!

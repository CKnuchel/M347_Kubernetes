## Erklärung der Mediawiki Konfigurationsdatei

### Inhaltsverzeichnis
- [Erklärung der Mediawiki Konfigurationsdatei](#erklärung-der-mediawiki-konfigurationsdatei)
  - [Inhaltsverzeichnis](#inhaltsverzeichnis)
  - [Einleitung](#einleitung)
  - [Anforderungen an das System](#anforderungen-an-das-system)
  - [Generelles](#generelles)
  - [Konfigurationsdatei](#konfigurationsdatei)
    - [Namespace](#namespace)
    - [PersistentVolume](#persistentvolume)
      - [PersistentVolume für das Mediawiki](#persistentvolume-für-das-mediawiki)
      - [PersistentVolume für die MySQL Datenbank](#persistentvolume-für-die-mysql-datenbank)
    - [PersistentVolumeClaim](#persistentvolumeclaim)
      - [PersistentVolumeClaim für das Mediawiki](#persistentvolumeclaim-für-das-mediawiki)
      - [PersistentVolumeClaim für die MySQL Datenbank](#persistentvolumeclaim-für-die-mysql-datenbank)
    - [ConfigMap](#configmap)
    - [Secret](#secret)
    - [Deployment](#deployment)
      - [Deployment für das Mediawiki](#deployment-für-das-mediawiki)
      - [Deployment für die MySQL Datenbank](#deployment-für-die-mysql-datenbank)
    - [Service](#service)
      - [Service für das Mediawiki](#service-für-das-mediawiki)
      - [Service für die MySQL Datenbank](#service-für-die-mysql-datenbank)
    - [NetworkPolicy](#networkpolicy)
  - [Fazit](#fazit)
- [Installation des Mediawikis](#installation-des-mediawikis)
  - [Installation auf einem Kubernetes Cluster](#installation-auf-einem-kubernetes-cluster)
  - [Aufsetzen des Mediawikis](#aufsetzen-des-mediawikis)
    - [Wichtige Konfigurationsschritte](#wichtige-konfigurationsschritte)
      - [Konfiguration der Datenbank](#konfiguration-der-datenbank)
      - [Kopieren der `LocalSettings.php` Datei](#kopieren-der-localsettingsphp-datei)
- [Testkonzept](#testkonzept)
  - [Testprotokoll](#testprotokoll)
- [Quellen und Referenzen](#quellen-und-referenzen)

### Einleitung
**Was ist Mediawiki:**
Mediawiki ist eine freie Software zur Dokumentation und Kollaboration. Es wird unter anderem von Wikipedia verwendet. Es ist in PHP geschrieben und verwendet eine MySQL Datenbank.

### Anforderungen an das System
- Einen Service welcher eine MySQL Datenbank bereitstellt
- Einen Service welcher Mediawiki bereitstellt

### Generelles
Wir haben uns für einen klaren Aufbau der Kubernetes Ressourcen entschieden. Aus diesem Grund haben wir für das Mediawiki einen eigenen Namespace erstellt. Dieser Namespace ist `mediawiki`.

### Konfigurationsdatei
Die Konfigurationsdatei für das Mediawiki ist in der Datei `mediawiki.yaml` zu finden. Diese Datei enthält alle notwendigen Ressourcen um das Mediawiki in Kubernetes zu deployen.

Übersicht der Ressourcen:
- Namespace
- PersistentVolume
- PersistentVolumeClaim
- ConfigMap
- Secret
- Deployment
- Service
- NetworkPolicy

#### Namespace
Der Namespace `mediawiki` wird erstellt um eine klare Trennung der Ressourcen zu gewährleisten.

Die Definition des Namespaces sieht wie folgt aus:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mediawiki
```

Hier wird der Namespace `mediawiki` erstellt. Dieser Namespace wird für alle weiteren Ressourcen verwendet und muss daher zuerst erstellt werden.

#### PersistentVolume
Das PersistentVolume wird erstellt um die Daten des Mediawikis persistent zu speichern. Dies ist notwendig um die Daten auch nach einem Neustart des Pods wiederherstellen zu können.

##### PersistentVolume für das Mediawiki
Die Definition des PersistentVolumes sieht wie folgt aus:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mediawiki-pv
  namespace: mediawiki
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 5Gi
  storageClassName: manual
  hostPath:
    path: /tmp/mediawiki-data
```

Hier wird ein PersistentVolume mit dem Namen `mediawiki-pv` erstellt. Dieses Volume hat eine Größe von 5GiB und wird auf dem Host unter `/tmp/mediawiki-data` gespeichert.

##### PersistentVolume für die MySQL Datenbank
Die Definition des PersistentVolumes sieht wie folgt aus:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mediawiki-sql-pv
  namespace: mediawiki
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 4Gi
  storageClassName: manual
  hostPath:
    path: /tmp/mediawiki-sql
```

Hier wird ein PersistentVolume mit dem Namen `mediawiki-sql-pv` erstellt. Dieses Volume hat eine Größe von 4GiB und wird auf dem Host unter `/tmp/mediawiki-sql` gespeichert.

#### PersistentVolumeClaim
Der PersistentVolumeClaim wird erstellt um das PersistentVolume an den Pod zu binden. Dieser Claim wird im Deployment verwendet um das Volume zu mounten.

##### PersistentVolumeClaim für das Mediawiki
Die Definition des PersistentVolumeClaims sieht wie folgt aus:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mediawiki-pv-claim
  namespace: mediawiki
spec:
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
```

Hier wird ein PersistentVolumeClaim mit dem Namen `mediawiki-pv-claim` erstellt. Dieser Claim bindet das PersistentVolume `mediawiki-pv` an den Pod. Die Zuweisung zum Pod erfolgt im Deployment.

##### PersistentVolumeClaim für die MySQL Datenbank
Die Definition des PersistentVolumeClaims sieht wie folgt aus:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mediawiki-sql-pv-claim
  namespace: mediawiki
spec:
  resources:
    requests:
      storage: 4Gi
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
```

Hier wird ein PersistentVolumeClaim mit dem Namen `mediawiki-sql-pv-claim` erstellt. Dieser Claim bindet das PersistentVolume `mediawiki-sql-pv` an den Pod. Die Zuweisung zum Pod erfolgt im Deployment.

#### ConfigMap
Die ConfigMap wird erstellt um die Konfiguration des Mediawikis zu speichern. Diese Konfiguration wird im Deployment verwendet um die Konfiguration in den Pod zu mounten. Dies entspricht sozuagen einem Satz an Variablen die im Pod verfügbar sind.

Die Definition der ConfigMap sieht wie folgt aus:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mediawiki-env
  namespace: mediawiki
data:
  MYSQL_DATABASE: wikidatabase
```

Hier wird eine ConfigMap mit dem Namen `mediawiki-env` erstellt. Diese ConfigMap enthält die Variable `MYSQL_DATABASE` mit dem Wert `wikidatabase`. Diese Variable wird im Deployment verwendet um die MySQL Datenbank zu konfigurieren.

#### Secret
Das Secret wird erstellt um die Zugangsdaten zur MySQL Datenbank zu speichern. Diese Zugangsdaten werden im Deployment verwendet um die MySQL Datenbank zu konfigurieren. Dies stellt im Pod die Umgebungsvariablen zur Verfügung.

Die Definition des Secrets sieht wie folgt aus:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mediawiki-mysql-secret
  namespace: mediawiki
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: UGFzc3cwcmQ= #Passw0rd
```

Hier wird ein Secret mit dem Namen `mediawiki-mysql-secret` erstellt. Dieses Secret enthält die Variable `MYSQL_ROOT_PASSWORD` mit dem Wert `Passw0rd`. Dieser Wert ist base64 kodiert.

Um den Wert zu kodieren kann folgender Befehl verwendet werden:
```bash
echo -n 'Passw0rd' | base64
```

#### Deployment
Ein Deployment wird erstellt um den Pod zu definieren. Dieser Pod enthält das zu verwendende Image, die Umgebungsvariablen, die Volumes und die Ports. 

##### Deployment für das Mediawiki
Die Definition des Deployments sieht wie folgt aus:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    service: mediawiki
  name: mediawiki-deployment
  namespace: mediawiki
spec:
  replicas: 1
  selector:
    matchLabels:
      service: mediawiki
  template:
    metadata:
      labels:
        network/wikinetwork: "true" # Define the network policy
        service: mediawiki
    spec:
      containers:
        - name: mediawiki
          image: mediawiki:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 80
          resources:
            limits:
              memory: "1Gi"
              cpu: "500m"
          env:
            - name: MEDIAWIKI_DB_HOST
              value: mediawiki-mysql
            - name: MEDIAWIKI_DB_NAME
              valueFrom:
                configMapKeyRef:
                  key: MYSQL_DATABASE
                  name: mediawiki-env
            - name: MEDIAWIKI_DB_USER
              value: root
            - name: MEDIAWIKI_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: MYSQL_ROOT_PASSWORD
                  name: mediawiki-mysql-secret
      restartPolicy: Always
```

Hier wird ein Deployment mit dem Namen `mediawiki-deployment` erstellt. Dieses Deployment erstellt einen Pod mit dem Image `mediawiki:latest`. Der Pod hat die Umgebungsvariablen `MEDIAWIKI_DB_HOST`, `MEDIAWIKI_DB_NAME`, `MEDIAWIKI_DB_USER` und `MEDIAWIKI_DB_PASSWORD`. Diese Variablen werden verwendet um die MySQL Datenbank zu konfigurieren. In dem Deployment ist auch die Zuweisung der Umgebungsvariablen aus der ConfigMap und dem Secret zu sehen.
Das Deployment erstellt einen Pod mit einem Container welcher auf Port 80 lauscht.
Weiter werden die Ressourcen für den Pod definiert. Der Pod hat eine Limitierung von 1GiB RAM und 500m CPU.

##### Deployment für die MySQL Datenbank
Die Definition des Deployments sieht wie folgt aus:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    service: mediawiki-mysql
  name: mediawiki-mysql
  namespace: mediawiki
spec:
  replicas: 1
  selector:
    matchLabels:
      service: mediawiki-mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        network/wikinetwork: "true"
        service: mediawiki-mysql
    spec:
      containers:
        - name: mediawiki-mysql
          image: mysql:8.0
          imagePullPolicy: Always
          resources:
            limits:
              memory: "1Gi"
              cpu: "500m"
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_DATABASE
              valueFrom:
                configMapKeyRef:
                  key: MYSQL_DATABASE
                  name: mediawiki-env
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: MYSQL_ROOT_PASSWORD
                  name: mediawiki-mysql-secret
      restartPolicy: Always
```

Hier wird ein Deployment mit dem Namen `mediawiki-mysql` erstellt. Dieses Deployment erstellt einen Pod mit dem Image `mysql:8.0`. Der Pod hat die Umgebungsvariablen `MYSQL_DATABASE` und `MYSQL_ROOT_PASSWORD`. Diese Variablen werden verwendet um die MySQL Datenbank zu konfigurieren.

#### Service
Ein Service wird erstellt um den Pod von aussen erreichbar zu machen. Dieser Service leitet die Anfragen an den Pod weiter.

##### Service für das Mediawiki
Die Definition des Services sieht wie folgt aus:
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    service: mediawiki
  name: mediawiki-service
  namespace: mediawiki
spec:
  type: NodePort
  ports:
    - name: "8000"
      port: 8000
      targetPort: 80
  selector:
    service: mediawiki
```

Hier wird ein Service mit dem Namen `mediawiki-service` erstellt. Dieser Service leitet die Anfragen auf Port 8000 an den Pod weiter. Der Service ist vom Typ `NodePort`, was bedeutet dass der Service auf jedem Node auf Port 8000 erreichbar ist.

##### Service für die MySQL Datenbank
Die Definition des Services sieht wie folgt aus:
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    service: mediawiki-mysql
  name: mediawiki-mysql
  namespace: mediawiki
spec:
  ports:
    - name: "33060"
      port: 33060
      targetPort: 3306
  selector:
    service: mediawiki-mysql
  type: ClusterIP
```

Hier wird ein Service mit dem Namen `mediawiki-mysql` erstellt. Dieser Service leitet die Anfragen auf Port 33060 an den Pod weiter. Der Service ist vom Typ `ClusterIP`, was bedeutet dass der Service nur innerhalb des Clusters erreichbar ist.

#### NetworkPolicy
Die NetworkPolicy wird erstellt um den Zugriff auf die Pods zu beschränken. Diese Policy wird verwendet um den Zugriff auf die Pods zu beschränken. In diesem Fall wird der Zugriff auf die Pods nur von dem Mediawiki Service erlaubt.

Die Definition der NetworkPolicy sieht wie folgt aus:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: wikinetwork
  namespace: mediawiki
spec:
  podSelector:
    matchLabels:
      network/wikinetwork: "true"
  ingress:
    - from:
        - podSelector:
            matchLabels:
              network/wikinetwork: "true"
```

Hier wird eine NetworkPolicy mit dem Namen `wikinetwork` erstellt. Diese Policy erlaubt den Zugriff auf die Pods nur von Pods welche das Label `network/wikinetwork: "true"` haben. Dieses Label wird in den Deployments definiert. Dieses haben wir lediglich für den Mediawiki Service definiert.

### Fazit
Die Definition des Mediawikis in Kubernetes ist sehr umfangreich. Es müssen viele Ressourcen erstellt werden um das Mediawiki zu deployen. Es ist jedoch sehr flexibel und kann einfach angepasst werden. Es ist auch möglich das Mediawiki in einem Helm Chart zu definieren. Dies würde die Definition vereinfachen und die Wartung erleichtern.

Ebenso wäre es möglich das ganze Deployment übersichtlicher zu gestalten in dem die Ressourcen in einzelne Dateien aufgeteilt werden. Dies würde die Wartung und das Verständnis erleichtern. Da wir aber nur ein Mediawiki deployen, haben wir uns für eine Datei entschieden.

## Installation des Mediawikis

### Installation auf einem Kubernetes Cluster
Das Mediawiki kann auf einem bestehenden Kubernetes Cluster installiert werden. Dazu muss die Datei `mediawiki.yaml` auf den Server kopiert werden. Anschliessend kann das Mediawiki mit folgendem Befehl installiert werden:
```bash
kubectl apply -f mediawiki.yaml
```

Hierbei werden alle Ressourcen erstellt und das Mediawiki wird auf dem Cluster installiert. Nach der Installation kann das Mediawiki über den Service `mediawiki-service` auf Port 8000 erreicht werden.

### Aufsetzen des Mediawikis
Nach der Installation des Mediawikis kann die Konfiguration des Mediawikis über die Weboberfläche durchgeführt werden. Dazu muss die Weboberfläche des Mediawikis aufgerufen werden. Dies kann über die IP Adresse des Clusters und dem Port 8000 erfolgen.

Die IP Adresse des Clusters kann über den Befehl `kubectl get nodes -o wide` herausgefunden werden. Anschliessend kann die IP Adresse und der Port 8000 im Browser aufgerufen werden.

Falls das Cluster über MiniKube betrieben wird, kann die IP Adresse über den Befehl `minikube service list` herausgefunden werden.

Nach dem Aufruf der Weboberfläche kann die Konfiguration des Mediawikis durchgeführt werden. Es wird empfohlen die Konfiguration durchzuführen bevor das Mediawiki produktiv genutzt wird.

#### Wichtige Konfigurationsschritte
In diesem Teil werden wichtige Konfigurationsschritte für das Mediawiki beschrieben. Welche sich nicht von alleine verstehen lassen.

##### Konfiguration der Datenbank
Bei der Konfiguration der Datenbank muss der Hostname der Datenbank angegeben werden. Dieser ist in unserem Fall `mediawiki-mysql.mediawiki.svc.cluster.local:33060`. Ebenso muss der Benutzername und das Passwort angegeben werden. Der Benutzername ist `root` und das Passwort ist `Passw0rd`.

Am Ende der Konfiguration wird die Datenbank initialisiert. Dies kann einige Zeit in Anspruch nehmen. 
Nach Abschluss der Konfiguration, wird eine `LocalSettings.php` Datei generiert. Diese Datei muss im Anschluss in den Pod kopiert werden. Dies kann über den Befehl `kubectl cp` erfolgen.

##### Kopieren der `LocalSettings.php` Datei
Stellen Sie sicher, das Sie sich im Verzeichnis befinden, in welchem sich die `LocalSettings.php` Datei befindet. Anschliessend kann die Datei in den Pod kopiert werden. Dies kann über den Befehl `kubectl cp` erfolgen.

```bash
kubectl cp LocalSettings.php mediawiki-deployment-<pod-id>:/var/www/html/LocalSettings.php
```

Hierbei muss `<pod-id>` durch die ID des Pods ersetzt werden. Diese kann über den Befehl `kubectl get pods` herausgefunden werden.

## Testkonzept
Das Testkonzept kann verwendet werden um das Mediawiki zu testen. Es enthält Testfälle welche durchgeführt werden können um das Mediawiki zu testen.

| **Testfall Nr.** | **Beschreibung**                           | **Testschritte**                                                                                                                                   | **Erwartetes Ergebnis**               | **Erhaltenes Ergebnis**               | **Status**         |
|-------------------|-------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------|----------------------------------------|--------------------|
| 1                 | Mediawiki ist über den Browser erreichbar | 1. Ermitteln der IP-Adresse des Clusters<br>2. Aufrufen der Adresse im Browser<br>3. Das Mediawiki sollte angezeigt werden                        | Mediawiki ist erreichbar              | [Hier das tatsächliche Ergebnis eintragen] | [Erfolgreich/Nicht erfolgreich] |
| 2                 | Es wird ein neuer Artikel erstellt        | 1. Anmelden am Mediawiki<br>2. Klicken auf `Neue Seite`<br>3. Eingeben des Titels und des Inhalts<br>4. Klicken auf `Seite speichern`<br>5. Überprüfen, ob der Artikel erstellt wurde | Der Artikel wurde erstellt            | [Hier das tatsächliche Ergebnis eintragen] | [Erfolgreich/Nicht erfolgreich] |
| 3                 | Ein bestehender Artikel wird bearbeitet   | 1. Anmelden am Mediawiki<br>2. Klicken auf `Bearbeiten`<br>3. Bearbeiten des Inhalts<br>4. Klicken auf `Seite speichern`<br>5. Überprüfen, ob der Artikel bearbeitet wurde           | Der Artikel wurde bearbeitet          | [Hier das tatsächliche Ergebnis eintragen] | [Erfolgreich/Nicht erfolgreich] |


### Testprotokoll
Die durchgeführten Testfälle gemäss dem obigen Testkonzept wurden erfolgreich durchgeführt. Das Mediawiki ist erreichbar und es können Artikel erstellt und bearbeitet werden. Das Mediawiki ist somit funktionsfähig und kann produktiv genutzt werden.

| **Testfall Nr.** | **Beschreibung**                           | **Testschritte**                                                                                                                                   | **Erwartetes Ergebnis**               | **Erhaltenes Ergebnis**               | **Status**         |
|-------------------|-------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------|----------------------------------------|--------------------|
| 1                 | Mediawiki ist über den Browser erreichbar | 1. Ermitteln der IP-Adresse des Clusters<br>2. Aufrufen der Adresse im Browser<br>3. Das Mediawiki sollte angezeigt werden                        | Mediawiki ist erreichbar              | Mediawiki ist erreichbar              | Erfolgreich        |
| 2                 | Es wird ein neuer Artikel erstellt        | 1. Anmelden am Mediawiki<br>2. Klicken auf `Neue Seite`<br>3. Eingeben des Titels und des Inhalts<br>4. Klicken auf `Seite speichern`<br>5. Überprüfen, ob der Artikel erstellt wurde | Der Artikel wurde erstellt            | Der Artikel wurde erstellt            | Erfolgreich        |
| 3                 | Ein bestehender Artikel wird bearbeitet   | 1. Anmelden am Mediawiki<br>2. Klicken auf `Bearbeiten`<br>3. Bearbeiten des Inhalts<br>4. Klicken auf `Seite speichern`<br>5. Überprüfen, ob der Artikel bearbeitet wurde           | Der Artikel wurde bearbeitet          | Der Artikel wurde bearbeitet          | Erfolgreich        |


## Quellen und Referenzen
- [Mediawiki](https://www.mediawiki.org/wiki/MediaWiki)
- [Kubernetes](https://kubernetes.io/)
- [Docker Hub](https://hub.docker.com/)

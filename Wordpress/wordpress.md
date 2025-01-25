# Bereitstellung von WordPress mit MySQL auf Kubernetes

## Inhaltsverzeichnis

- [Einleitung](#einleitung)
- [Anforderungen](#anforderungen)
- [Infrastruktur](#infrastruktur)
- [Konfiguration](#konfiguration)
- [Testplan](#testplan)
- [Installationsanleitung](#installationsanleitung)
- [Hilfestellungen](#hilfestellungen)
- [Quellen und Referenzen](#quellenundreferenzen)

## Einleitung

**Was ist WordPress:**
WordPress ist ein freies Content Management System (CMS), das zur Erstellung und Verwaltung von Websites verwendet wird.Es nutzt eine MySQL Datenbank zur Speicherung von Inhalten. Dieses Projekt beschreibt die Installation, Konfiguration und Tests eines Wordpress Systems auf einem Kubernetes Cluster.

## Anforderungen

- Einen Service, der eine MySQL Datenbank bereitstellt.
- Einen Service, der WordPress bereitstellt.
- Installation Hands-off: Das System soll weitgehend automatisch konfiguriert und installiert werden können. 
- Komponenten: Bereitstellung von Wordpress und einer MySQL Datenbank.

## Infrastruktur

### Visuelles Diagramm

Die Infrastruktur unseres Systems umfasst folgende Komponenten:

- WordPress Pod: Hostet die WordPress Anwendung.
- MySQL Pod: Hostet die MySQL Datenbank.
- PersistentVolumes: Speichert Daten persistent sowohl für WordPress als auch MySQL.
- Services: Erlauben die Kommunikation zwischen den Pods und der Aussenwelt.

Das folgende Diagramm visualisiert die Verbindungen:

![Diagramm](./images/Wordpress_Infrastruktur.svg)

## Konfiguration

### Secret für MySQL

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-pass
  namespace: wordpress
type: Opaque
data:
  password: password
```

### MySQL Komponenten

#### MySQL Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql-svc
  labels:
    app: wordpress
  namespace: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
```

#### MySQL PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  labels:
    app: wordpress
  namespace: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

#### MySQL Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql-deployment
  labels:
    app: wordpress
  namespace: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```

### WordPress-Komponenten

#### WordPress-Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress-svc
  labels:
    app: wordpress
  namespace: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
  type: LoadBalancer
```

#### WordPress PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pv-claim
  labels:
    app: wordpress
  namespace: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

#### WordPress Deployment

```yaml
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: wordpress-deployment
  labels:
    app: wordpress
  namespace: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:4.8-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql-svc
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wordpress-pv-claim
```

# Testplan

## Testfälle

| **Testfall Nr.** | **Beschreibung**                    | **Testschritte**                                                                                       | **Erwartetes Ergebnis**                  |
|-------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------|------------------------------------------|
| 1                 | WordPress ist über den Browser erreichbar | 1. Starte den Kubernetes-Cluster. 2. Wende die Konfiguration mit kubectl apply an. <br> 2. Rufe die URL `http://<Server-IP>:<Port>` im Browser auf.               | WordPress Startseite wird geladen        |
| 2                 | Neues Benutzerkonto erstellen      | 1. Melde dich im WordPress-Admin-Dashboard an. <br> 2. Gehe zu "Benutzer" > "Neu hinzufügen". <br> 3. Fülle die Felder aus und speichere. | Benutzerkonto wird erstellt             |
| 3                 | Blogbeitrag erstellen              | 1. Melde dich im WordPress-Admin-Dashboard an. <br> 2. Gehe zu "Beiträge" > "Erstellen". <br> 3. Fülle die Felder aus und veröffentliche den Beitrag. | Blogbeitrag wird korrekt veröffentlicht |
| 4                 | Plugin installieren                | 1. Melde dich im WordPress-Admin-Dashboard an. <br> 2. Gehe zu "Plugins" > "Installieren". <br> 3. Suche ein Plugin und installiere es. | Plugin wird erfolgreich installiert     |

## Testprotokoll

| **Testfall Nr.** | **Beschreibung**                    | **Testschritte**                                                                                       | **Erwartetes Ergebnis**                  | **Erhaltenes Ergebnis**                  | **Status**     |
|-------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------|------------------------------------------|------------------------------------------|----------------|
| 1                 | WordPress ist über den Browser erreichbar | 1. Starte den Kubernetes-Cluster. 2. Wende die Konfiguration mit kubectl apply an. <br> 3. Rufe die URL `http://<Server-IP>:<Port>` im Browser auf.               | WordPress Startseite wird geladen        | WordPress Startseite wurde geladen       | Erfolgreich    |
| 2                 | Neues Benutzerkonto erstellen      | 1. Melde dich im WordPress-Admin-Dashboard an. <br> 2. Gehe zu "Benutzer" > "Neu hinzufügen". <br> 3. Fülle die Felder aus und speichere. | Benutzerkonto wird erstellt             | Benutzerkonto wurde erfolgreich erstellt | Erfolgreich    |
| 3                 | Blogbeitrag erstellen              | 1. Melde dich im WordPress-Admin-Dashboard an. <br> 2. Gehe zu "Beiträge" > "Erstellen". <br> 3. Fülle die Felder aus und veröffentliche den Beitrag. | Blogbeitrag wird korrekt veröffentlicht | Blogbeitrag wurde korrekt veröffentlicht | Erfolgreich    |
| 4                 | Plugin installieren                | 1. Melde dich im WordPress-Admin-Dashboard an. <br> 2. Gehe zu "Plugins" > "Installieren". <br> 3. Suche ein Plugin und installiere es. | Plugin wird erfolgreich installiert     | Plugin wurde erfolgreich installiert     | Erfolgreich    |


## Installationsanleitung

### Via GitHub

  1. Klonen des GitHub Repository "git clone <URL des Repository>"
  2. In das korrekte Verzeichnis wechseln cd ...
  3. Ausführen von "kubectl apply -f ./" 
  4. Korrekte IP Adresse ermitteln " minikube service wordpress --url -n wordpress" 
  5. Navigieren zur IP Adresse des LoadBalancer Services.

## Hilfestellungen

- Internetrecherche, Kubernetes und WordPress Dokumentationen.

## Quellen und Referenzen

- [Kubernetes-Dokumentation](https://kubernetes.io)
- [WordPress-Dokumentation](https://wordpress.org)


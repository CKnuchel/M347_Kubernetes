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
WordPress ist ein freies Content Management System (CMS), das zur Erstellung und Verwaltung von Websites verwendet wird. Es ist in PHP geschrieben und nutzt eine MySQL Datenbank zur Speicherung von Inhalten.

## Anforderungen

- Einen Service, der eine MySQL Datenbank bereitstellt.
- Einen Service, der WordPress bereitstellt.

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

## Testplan

### Testfälle

| **Testfall**         | **Beschreibung**                              | **Erwartetes Ergebnis**            |
|----------------------|----------------------------------------------|------------------------------------|
| WordPress erreichbar | Zugriff auf die Startseite von WordPress     | WordPress Startseite wird geladen |
| Datenbankverbindung  | Verbindung zur MySQL Datenbank herstellen    | Verbindung erfolgreich            |

### Ergebnisse

| **Testfall**         | **Status**     |
|----------------------|----------------|
| WordPress erreichbar | Erfolgreich    |
| Datenbankverbindung  | Erfolgreich    |

## Installationsanleitung

### Via GitHub

  1. Klonen des GitHub Repository "git clone <URL des Repository>"
  2. In das korrekte Verzeichnis wechseln cd ...
  3. Ausführen von "kubectl apply -f ./" 
  4. Korrekte IP Adresse ermitteln " minikube service wordpress --url -n wordpress" 
  5. Navigieren zur IP Adresse des LoadBalancer Services.
  6. Eingabe der Datenbankinformationen:
      - Datenbank Host: wordpress-mysql
      - Benutzer: root
      - Passwort: (wie im Secret definiert)


## Hilfestellungen

- Internetrecherche, Kubernetes und WordPress Dokumentationen.

## Quellen und Referenzen

- [Kubernetes-Dokumentation](https://kubernetes.io)
- [WordPress-Dokumentation](https://wordpress.org)


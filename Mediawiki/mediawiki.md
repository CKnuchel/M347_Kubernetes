# Projektdokumentation: MediaWiki

## Inhaltsverzeichnis
- [Einleitung](#einleitung)
- [Anforderungen](#anforderungen)
- [Infrastruktur](#infrastruktur)
- [Konfiguration](#konfiguration)
- [Testplan](#testplan)
- [Testprotokoll](#testprotokoll)
- [Installationsanleitung](#installationsanleitung)
- [Hilfestellungen](#hilfestellungen)

---

## Einleitung

MediaWiki ist eine Open-Source-Software für Dokumentation und Zusammenarbeit. Dieses Projekt dokumentiert die Installation, Konfiguration und Tests eines MediaWiki-Systems auf einem Kubernetes-Cluster.

## Anforderungen

### Beschreibung
- **Installation Hands-off**: Das System soll weitgehend automatisch konfiguriert und installiert werden können. Aus diesem Grund wurde alles in einer YAML-Datei zusammengefasst.
- **Komponenten**: Die Bereitstellung umfasst eine MySQL-Datenbank und die MediaWiki-Anwendung.

---

## Infrastruktur

### Systemübersicht
Das folgende Diagramm zeigt die Komponenten des Systems und deren Verbindungen:

- **MediaWiki Deployment**: Stellt die Webanwendung bereit.
- **MySQL Deployment**: Dient als Datenbank für MediaWiki.
- **Persistent Volumes**: Speichert MediaWiki-Daten und SQL-Daten persistent.
- **Services**: Verbinden die Deployments und ermöglichen den Zugriff.
- **NetworkPolicy**: Regelt die Kommunikation zwischen den Pods.

<br>

![Systemübersicht](images/mediawiki_infra.svg)

---

## Konfiguration

### Kubernetes Ressourcen
Die Konfigurationsdateien in `mediawiki.yml` enthalten die folgenden Ressourcen:

#### Namespace
Der Namespace sorgt für eine Trennung der Ressourcen:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mediawiki
```

#### PersistentVolumes
Speichern die Daten persistent:

**MediaWiki-Daten:**
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

**MySQL-Daten:**
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

#### PersistentVolumeClaims
Binden die PersistentVolumes an die Pods:

**MediaWiki-Daten:**
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

**MySQL-Daten:**
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

#### ConfigMap
Speichert Konfigurationsvariablen:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mediawiki-env
  namespace: mediawiki
data:
  MYSQL_DATABASE: wikidatabase
```

#### Secrets
Speichert Passwörter und sensible Daten:
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

#### Deployments
Definieren die Pods für MediaWiki und MySQL:

**MediaWiki Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
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
        service: mediawiki
    spec:
      containers:
      - name: mediawiki
        image: mediawiki:latest
        env:
        - name: MEDIAWIKI_DB_HOST
          value: mediawiki-mysql
        - name: MEDIAWIKI_DB_NAME
          valueFrom:
            configMapKeyRef:
              key: MYSQL_DATABASE
              name: mediawiki-env
        - name: MEDIAWIKI_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              key: MYSQL_ROOT_PASSWORD
              name: mediawiki-mysql-secret
```

**MySQL Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mediawiki-mysql
  namespace: mediawiki
spec:
  replicas: 1
  selector:
    matchLabels:
      service: mediawiki-mysql
  template:
    metadata:
      labels:
        service: mediawiki-mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
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
```

#### Services
Ermöglichen die Kommunikation zwischen den Pods und externem Zugriff:

**MediaWiki Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mediawiki-service
  namespace: mediawiki
spec:
  type: NodePort
  ports:
    - port: 8000
      targetPort: 80
  selector:
    service: mediawiki
```

**MySQL Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mediawiki-mysql
  namespace: mediawiki
spec:
  ports:
    - port: 33060
      targetPort: 3306
  selector:
    service: mediawiki-mysql
```

---

## Testplan

### Testfälle

| **Testfall Nr.** | **Beschreibung**                           | **Testschritte**                                                                                  | **Erwartetes Ergebnis**         |
|-------------------|-------------------------------------------|---------------------------------------------------------------------------------------------------|---------------------------------|
| 1                 | MediaWiki ist über den Browser erreichbar | 1. Starte den Kubernetes-Cluster.<br>2. Wende die Konfiguration mit `kubectl apply` an.<br>3. Rufe die URL `http://<Cluster-IP>:8000` im Browser auf.  | MediaWiki wird angezeigt.      |
| 2                 | Neuer Artikel wird erstellt               | 1. Melde dich im MediaWiki an.<br>2. Klicke auf "Neue Seite".<br>3. Fülle Titel und Inhalt aus.<br>4. Speichere die Seite. | Artikel wird korrekt angezeigt.|
| 3                 | Artikel bearbeiten                        | 1. Melde dich im MediaWiki an.<br>2. Öffne einen bestehenden Artikel.<br>3. Bearbeite den Inhalt.<br>4. Speichere die Änderungen. | Änderungen sind sichtbar.      |

---

## Testprotokoll

| **Testfall Nr.** | **Beschreibung**                           | **Testschritte**                                                                                  | **Erwartetes Ergebnis**         | **Erhaltenes Ergebnis**               | **Status**         |
|-------------------|-------------------------------------------|---------------------------------------------------------------------------------------------------|---------------------------------|---------------------------------------|--------------------|
| 1                 | MediaWiki ist über den Browser erreichbar | 1. Starte den Kubernetes-Cluster.<br>2. Wende die Konfiguration mit `kubectl apply` an.<br>3. Rufe die URL `http://<Cluster-IP>:8000` im Browser auf.  | MediaWiki wird angezeigt.      | MediaWiki wird angezeigt.             | Erfolgreich        |
| 2                 | Neuer Artikel wird erstellt               | 1. Melde dich im MediaWiki an.<br>2. Klicke auf "Neue Seite".<br>3. Fülle Titel und Inhalt aus.<br>4. Speichere die Seite. | Artikel wird korrekt angezeigt. | Artikel wurde korrekt angezeigt.      | Erfolgreich        |
| 3                 | Artikel bearbeiten                        | 1. Melde dich im MediaWiki an.<br>2. Öffne einen bestehenden Artikel.<br>3. Bearbeite den Inhalt.<br>4. Speichere die Änderungen. | Änderungen sind sichtbar.      | Änderungen waren sichtbar.            | Erfolgreich        |

---

## Installationsanleitung

1. **Datei bereitstellen:**
   - Die Datei `mediawiki.yml` auf den Cluster laden.
2. **Ressourcen erstellen:**
   ```bash
   kubectl apply -f mediawiki.yml
   ```
3. **Weboberfläche aufrufen:**
   - Die Adresse `http://<Cluster-IP>:8000` im Browser öffnen.
4. **Datenbank konfigurieren:**
   - Host: `mediawiki-mysql.mediawiki.svc.cluster.local:33060`
   - Benutzer: `root`
   - Passwort: `Passw0rd`
5. **`LocalSettings.php` kopieren:**
   ```bash
   kubectl cp LocalSettings.php mediawiki-deployment-<pod-id>:/var/www/html/LocalSettings.php
   ```

---

## Hilfestellungen

### Mitlernende und Dozenten
- Unterstützung beim Verständnis von Kubernetes-Ressourcen.

### Internetrecherchen
- [Kubernetes-Dokumentation](https://kubernetes.io/docs/)
- [MediaWiki-Dokumentation](https://www.mediawiki.org/wiki/MediaWiki)
- [Docker Hub](https://hub.docker.com/)

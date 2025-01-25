# Projektdokumentation: Jira

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

Jira ist eine Softwarelösung für Projektmanagement, Aufgabenverfolgung und Team-Kollaboration. Dieses Projekt beschreibt die Installation, Konfiguration und Tests eines Jira-Systems auf einem Kubernetes-Cluster.

## Anforderungen

### Beschreibung
- **Installation Hands-off**: Das System soll weitgehend automatisch konfiguriert und installiert werden können. Aus diesem Grund wurde alles in einer YAML-Datei zusammengefasst.
- **Komponenten**: Bereitstellung von Jira und einer PostgreSQL-Datenbank.

---

## Infrastruktur

### Systemübersicht
Das folgende Diagramm sollte die Komponenten des Systems und deren Verbindungen visualisieren:

- **Jira Deployment**: Stellt die Jira-Anwendung bereit.
- **PostgreSQL Deployment**: Dient als Datenbank für Jira.
- **Persistent Volumes**: Speichern Jira-Daten und PostgreSQL-Daten persistent.
- **Services**: Verbinden Deployments und ermöglichen Zugriff.
- **NetworkPolicy**: Regelt die Kommunikation zwischen den Pods.

<br>

![Systemübersicht](images/jira_infra.svg)

---

## Konfiguration

### Kubernetes Ressourcen
Die YAML-Datei `jira.yml` enthält die folgenden Ressourcen:

#### Namespace
Trennt die Ressourcen des Jira-Systems:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: jira
```

#### PersistentVolumes
Speichern die Daten persistent:

**Jira-Daten:**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jira-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data/jira
```

**PostgreSQL-Daten:**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jira-db-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data/jira-db
```

#### PersistentVolumeClaims
Binden die PersistentVolumes an die Pods:

**Jira-Daten:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jira-pvc
  namespace: jira
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

**PostgreSQL-Daten:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jira-db-pvc
  namespace: jira
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

#### ConfigMap
Speichert Konfigurationsvariablen:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: jira-config
  namespace: jira
data:
  JIRA_HOME: /var/atlassian/application-data/jira
  POSTGRES_DB: jira
  POSTGRES_USER: jira
```

#### Secrets
Speichern Passwörter und sensible Daten:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: jira-secret
  namespace: jira
type: Opaque
data:
  db-password: UGFzc3cwcmQ= #Passw0rd
```

#### Deployments
Definieren die Pods für Jira und PostgreSQL:

**Jira Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jira
  namespace: jira
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jira
  template:
    metadata:
      labels:
        app: jira
    spec:
      containers:
        - name: jira
          image: atlassian/jira-software:latest
          resources:
            requests:
              memory: "2Gi"
              cpu: "1"
            limits:
              memory: "4Gi"
              cpu: "2"
          ports:
            - containerPort: 8080
          env:
            - name: JIRA_HOME
              valueFrom:
                configMapKeyRef:
                  name: jira-config
                  key: JIRA_HOME
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: jira-secret
                  key: db-password
          volumeMounts:
            - mountPath: /var/atlassian/application-data/jira
              name: jira-volume
      volumes:
        - name: jira-volume
          persistentVolumeClaim:
            claimName: jira-pvc
```

**PostgreSQL Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jira-db
  namespace: jira
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jira-db
  template:
    metadata:
      labels:
        app: jira-db
    spec:
      containers:
        - name: jira-db
          image: postgres:latest
          resources:
            requests:
              memory: "1Gi"
              cpu: "0.5"
            limits:
              memory: "2Gi"
              cpu: "1"
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: jira-config
                  key: POSTGRES_DB
            - name: POSTGRES_USER
              valueFrom:
                configMapKeyRef:
                  name: jira-config
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: jira-secret
                  key: db-password
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: jira-db-volume
      volumes:
        - name: jira-db-volume
          persistentVolumeClaim:
            claimName: jira-db-pvc
```

#### Services
Ermöglichen die Kommunikation zwischen den Pods und externem Zugriff:

**Jira Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: jira
  namespace: jira
spec:
  type: NodePort
  selector:
    app: jira
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

**PostgreSQL Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: jira-db
  namespace: jira
spec:
  selector:
    app: jira-db
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
```

#### NetworkPolicy
Regelt die Kommunikation zwischen Jira und der Datenbank:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-jira-db
  namespace: jira
spec:
  podSelector:
    matchLabels:
      app: jira
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: jira-db
      ports:
        - protocol: TCP
          port: 5432
```

---

## Testplan

### Testfälle

| **Testfall Nr.** | **Beschreibung**                           | **Testschritte**                                                                                  | **Erwartetes Ergebnis**         |
|-------------------|-------------------------------------------|---------------------------------------------------------------------------------------------------|---------------------------------|
| 1                 | Jira ist über den Browser erreichbar      | 1. Starte den Kubernetes-Cluster.<br>2. Wende die Konfiguration mit `kubectl apply` an.<br>3. Rufe die URL `http://<Cluster-IP>:<Port>` im Browser auf.  | Jira wird angezeigt.           |
| 2                 | Projekt erstellen                        | 1. Melde dich bei Jira an.<br>2. Gehe zu "Projekte".<br>3. Klicke auf "Neues Projekt".<br>4. Fülle die Felder aus und speichere.  | Das Projekt wird erstellt.     |
| 3                 | Aufgaben erstellen und zuweisen          | 1. Öffne ein Projekt.<br>2. Klicke auf "Neue Aufgabe".<br>3. Weise die Aufgabe einem Benutzer zu und speichere. | Aufgabe wird korrekt zugewiesen.|

---

## Testprotokoll

| **Testfall Nr.** | **Beschreibung**                           | **Testschritte**                                                                                  | **Erwartetes Ergebnis**         | **Erhaltenes Ergebnis**               | **Status**         |
|-------------------|-------------------------------------------|---------------------------------------------------------------------------------------------------|---------------------------------|---------------------------------------|--------------------|
| 1                 | Jira ist über den Browser erreichbar      | 1. Starte den Kubernetes-Cluster.<br>2. Wende die Konfiguration mit `kubectl apply` an.<br>3. Rufe die URL `http://<Cluster-IP>:<Port>` im Browser auf.  | Jira wird angezeigt.           | Jira wird angezeigt.                  | Erfolgreich        |
| 2                 | Projekt erstellen                        | 1. Melde dich bei Jira an.<br>2. Gehe zu "Projekte".<br>3. Klicke auf "Neues Projekt".<br>4. Fülle die Felder aus und speichere.  | Das Projekt wird erstellt.     | Projekt wurde erfolgreich erstellt.   | Erfolgreich        |
| 3                 | Aufgaben erstellen und zuweisen          | 1. Öffne ein Projekt.<br>2. Klicke auf "Neue Aufgabe".<br>3. Weise die Aufgabe einem Benutzer zu und speichere. | Aufgabe wird korrekt zugewiesen.| Aufgabe wurde korrekt zugewiesen.     | Erfolgreich        |

---

## Installationsanleitung

1. **Datei bereitstellen:**
   - Die Datei `jira.yml` auf den Cluster laden.
2. **Ressourcen erstellen:**
   ```bash
   kubectl apply -f jira.yml
   ```
3. **Weboberfläche aufrufen:**
   - Die IP und den Port des Jira-Services finden:
     ```bash
     miniube service jira -n jira --url
     ```
   - Die Adresse `http://<Cluster-IP>:<Port>` im Browser öffnen.

4. **Konfiguration durchführen:**
   - Database-Setup durchführen und die Datenbankverbindung konfigurieren.

      - **Datenbanktyp:** PostgreSQL
      - **Datenbankhost:** jira-db-svc
      - **Datenbankport:** 5432
      - **Datenbankname:** jira (aus ConfigMap)
      - **Benutzername:** jira (aus ConfigMap)
      - **Passwort:** Passw0rd (aus Secret)
      - **Schema:** public

    - Testen der Verbindung mit dem Button "Test Connection".
    - Weiter mit "Next".

5. **Application Properties setzen:**
   - **Application Title:** Jira -> Dieser Name wird im Browser-Tab angezeigt.
   - **Mode:** Public -> Jira ist für alle Benutzer sichtbar.
   - **Base URL:** Die URL, unter der Jira erreichbar ist. Diese wird standardmäßig korrekt eingestellt.
  
6. **Lizenzschlüssel eingeben:**
    - Erstellen Sie ein Konto auf der [Atlassian-Website](https://www.atlassian.com/software/jira).
    - Melden Sie sich an und gehen Sie zu "My Atlassian".
    - Wählen Sie "New Evaluation License" und füllen Sie das Formular aus.
    - Kopieren Sie den Lizenzschlüssel und fügen Sie ihn in das Feld ein.

7. **Erstellen Sie einen Administrator-Account:**
    - Füllen Sie die Felder für den Administrator aus und klicken Sie auf "Next".

8. **Mailserver konfigurieren:**
    - Wählen Sie "Skip" oder konfigurieren Sie den Mailserver.

9. **Fertigstellung:**
    - Wählen Sie die gewünschte Sprache aus und klicken Sie auf "Finish".
  
Jira ist nun einsatzbereit und kann für die Projektverwaltung genutzt werden.

---

## Hilfestellungen

### Mitlernende und Dozenten
- Unterstützung bei der Konfiguration und Einrichtung von Jira.

### Internetrecherchen
- [Jira-Dokumentation](https://www.atlassian.com/software/jira)
- [Kubernetes-Dokumentation](https://kubernetes.io/docs/)
- [Docker Hub](https://hub.docker.com/)

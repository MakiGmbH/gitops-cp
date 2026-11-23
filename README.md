# Maki Sandbox – GitOps

Die GitOps-Variante erweitert den lokalen Kubernetes-Cluster um **Kustomize** und **ArgoCD**. Kustomize fasst die Kubernetes-Manifeste zusammen und erzeugt die ConfigMap aus der HTML-Datei. ArgoCD kann anschließend das Git-Repository regelmäßig mit dem lokalen Cluster abgleichen.

Die Variante ist für Nutzer gedacht, die neben Docker und Kubernetes auch deklarative Konfiguration, Versionsverwaltung und automatisierte Bereitstellung praktisch kennenlernen möchten.

## Funktionen

- Lokaler Kubernetes-Cluster mit k3d
- Drei NGINX-Replikate im Deployment
- Ressourcenanforderungen und -limits für CPU und Arbeitsspeicher
- Liveness- und Readiness-Probes zur Zustandsprüfung
- Kustomize zur Zusammenführung der Manifeste und Generierung der HTML-ConfigMap
- ArgoCD Application für den Abgleich eines Git-Repositories mit dem Cluster
- Automatische Synchronisierung sowie Self-Healing und Pruning durch die konfigurierte ArgoCD-Sync-Policy

## Voraussetzungen

| Komponente | Mindestanforderung / Hinweis |
|---|---|
| Betriebssystem | macOS (Intel oder Apple Silicon), Windows 11 oder eine gängige 64-Bit-Linux-Distribution |
| Arbeitsspeicher | 8 GB; 16 GB empfohlen |
| Prozessor | 4 CPU-Kerne |
| Speicherplatz | 15 GB frei; 25 GB empfohlen |
| Software | Docker, Git, k3d, kubectl und ein Terminal |
| Rechte | Administrator- bzw. `sudo`-Rechte für die Installation |
| Internet | Für das Klonen, den erstmaligen Image-Download und den Git-Abgleich durch ArgoCD |

Unter Windows wird Docker Desktop mit aktivierter WSL2-Integration empfohlen. k3d benötigt eine funktionierende Docker-Laufzeitumgebung. Für den automatischen Abgleich muss ArgoCD das konfigurierte Git-Repository erreichen können.

## Installation der Voraussetzungen

Installieren Sie Docker, Git, k3d und kubectl und prüfen Sie anschließend:

```bash
docker --version
docker compose version
git --version
k3d version
kubectl version --client
```

Weiterführende Anleitungen:

- [Docker Desktop](https://docs.docker.com/desktop/)
- [Docker Engine](https://docs.docker.com/engine/install/)
- [k3d](https://k3d.io/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Kustomize](https://kubectl.docs.kubernetes.io/references/kustomize/)
- [ArgoCD](https://argo-cd.readthedocs.io/)
- [Git](https://git-scm.com/doc)

### Hinweise zu den Betriebssystemen

- **macOS:** Docker Desktop starten; k3d, kubectl und Git können beispielsweise über Homebrew installiert werden.
- **Windows 11:** WSL2 und Virtualisierung aktivieren, Docker Desktop installieren und die gewünschte WSL-Distribution unter *Settings → Resources → WSL Integration* aktivieren.
- **Linux:** Docker Engine installieren und sicherstellen, dass der eigene Benutzer Docker verwenden darf. Nach einer Änderung der Docker-Gruppenzugehörigkeit ist eine neue Anmeldung erforderlich.

## Sandbox starten

Klonen Sie das Repository und wechseln Sie in das Projektverzeichnis:

```bash
git clone https://github.com/MakiGmbH/maki-sandbox-gitops.git
cd maki-sandbox-gitops
```

Erstellen Sie den lokalen Kubernetes-Cluster:

```bash
k3d cluster create maki-sandbox -p "127.0.0.1:8080:80@loadbalancer" --agents 1 --api-port 127.0.0.1:6443
```

Wenden Sie die Ressourcen zunächst lokal mit Kustomize an:

```bash
kubectl apply -k kubernetes/
```

Die Portbindung auf `127.0.0.1` beschränkt den Zugriff auf den lokalen Rechner. Wenn Sie den Webserver bewusst im lokalen Netzwerk bereitstellen möchten, passen Sie die Host-IP-Bindung entsprechend an.

## Betrieb überprüfen

```bash
kubectl get nodes
kubectl get pods
kubectl get deployments,services,ingress
```

Die Webserver-Pods sollten den Status `Running` anzeigen. Öffnen Sie die Testseite im Browser:

**http://localhost:8080**

Beim ersten Start kann es einige Minuten dauern, bis alle Images geladen und die Pods bereit sind. Eine vorübergehende `404`-Meldung während des Starts ist möglich.

## Wie die GitOps-Automatisierung funktioniert

Die Automatisierung besteht aus drei Teilen:

1. **Git:** Das Repository enthält den gewünschten Zustand der Anwendung und die Kubernetes-Konfiguration.
2. **Kustomize:** `kubernetes/kustomization.yaml` fasst die Manifeste zusammen und erzeugt aus `kubernetes/anwendungen/index.html` eine ConfigMap. Änderungen am HTML-Inhalt führen zu einer geänderten ConfigMap-Kennung.
3. **ArgoCD:** Die Datei `argocd-app.yaml` beschreibt, welches Repository und welcher Pfad überwacht werden. ArgoCD gleicht den Cluster mit diesem Zustand ab.

**Wichtig:** Der k3d-Cluster und die Kubernetes-Ressourcen werden durch die Startbefehle erstellt. ArgoCD selbst wird separat installiert und mit dem Repository verbunden. Ohne diesen zusätzlichen Schritt funktioniert die Bereitstellung weiterhin manuell über `kubectl apply -k kubernetes/`; der automatische Git-Abgleich ist dann jedoch nicht aktiv.

## ArgoCD installieren und verbinden

### 1. ArgoCD im Cluster installieren

```bash
kubectl create namespace argocd
kubectl apply --server-side --force-conflicts -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Die Installation benötigt eine Internetverbindung. Warten Sie, bis die ArgoCD-Pods laufen:

```bash
kubectl get pods -n argocd -w
```

Beenden Sie die fortlaufende Anzeige mit `Strg+C`, sobald die Pods bereit sind.

### 2. Eigenes Git-Repository eintragen

Die mitgelieferte Datei `argocd-app.yaml` verweist standardmäßig auf das Repository `MakiGmbH/maki-sandbox-gitops`. Wenn Sie diese `maki-sandbox-gitops`-Kopie in Ihrem eigenen GitHub-Repository verwalten, ändern Sie vor dem Anwenden die Zeile `repoURL:` in `argocd-app.yaml` auf die URL Ihres eigenen Repositorys, beispielsweise:

```yaml
repoURL: 'https://github.com/MakiGmbH/maki-sandbox-gitops.git'
```

Wenn Sie die Dateien in ein anderes Repository übertragen, tragen Sie dessen tatsächliche URL ein. Die Änderungen an `argocd-app.yaml` müssen ebenfalls in Git gespeichert und hochgeladen sein.

Wenden Sie danach die Application an:

```bash
kubectl apply -f argocd-app.yaml
```

Ein öffentliches Repository kann ArgoCD ohne zusätzliche Git-Zugangsdaten abrufen. Für ein privates Repository müssen die Zugangsdaten in ArgoCD unter **Settings → Repositories → Connect Repo** hinterlegt werden. Verwenden Sie dafür einen geeigneten Personal Access Token (PAT) und speichern Sie ihn nicht in YAML-Dateien.

### 3. ArgoCD-Weboberfläche öffnen

Starten Sie die lokale Portweiterleitung und lassen Sie dieses Terminal geöffnet:

```bash
kubectl port-forward svc/argocd-server -n argocd 8081:443
```

Öffnen Sie anschließend:

**https://localhost:8081**

Das initiale Passwort lesen Sie in einem zweiten Terminal aus.

macOS, Linux oder WSL:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Windows PowerShell:

```powershell
$encoded = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($encoded))
```

Melden Sie sich mit dem Benutzernamen `admin` und dem ausgelesenen Passwort an. Der Browser weist auf das selbstsignierte Zertifikat hin; bestätigen Sie die Warnung für den lokalen Testzugriff.

ArgoCD fragt das Repository regelmäßig ab. Änderungen können deshalb mit etwas Verzögerung im Cluster ankommen. Über **Refresh** lässt sich die Synchronisierung in der Oberfläche anstoßen.

### Verhalten der automatischen Synchronisierung

Die mitgelieferte Application verwendet `automated`, `prune` und `selfHeal`:

- Änderungen im überwachten Git-Repository werden automatisch angewendet.
- Ressourcen, die aus der Git-Konfiguration entfernt wurden, können durch `prune` auch im Cluster gelöscht werden.
- Manuelle Änderungen direkt im Cluster können durch `selfHeal` wieder auf den in Git gespeicherten Zustand zurückgesetzt werden.

Git ist damit die maßgebliche Quelle für den gewünschten Zustand. Nehmen Sie dauerhafte Änderungen deshalb im Repository vor und übertragen Sie sie mit Git.

## Eigene Inhalte und Anwendungen

### Beispielseite ändern

Bearbeiten Sie die Datei:

`kubernetes/anwendungen/index.html`

Speichern und übertragen Sie die Änderung:

```bash
git status
git add kubernetes/anwendungen/index.html
git commit -m "Beispielseite angepasst"
git push origin main
```

Kustomize erzeugt aus der geänderten Datei eine neue ConfigMap-Kennung. ArgoCD erkennt die Änderung im Repository und synchronisiert die Kubernetes-Ressourcen entsprechend. Wenn ArgoCD nicht eingerichtet ist, wenden Sie die Änderungen manuell an:

```bash
kubectl apply -k kubernetes/
```

### Eigenes Container-Image verwenden

Passen Sie in `kubernetes/deployment.yaml` den Wert `image:` an das gewünschte Image an. Wenn Ihre Anwendung nicht auf Port `80` lauscht, müssen `containerPort` sowie die Werte `port` und `targetPort` im Service zueinander passen. Bei aktiver ArgoCD-Synchronisierung speichern und pushen Sie die Änderung; ohne ArgoCD wenden Sie sie mit `kubectl apply -k kubernetes/` an.

## Eigenes GitHub-Repository verwenden

Wenn Sie nicht in ein eigenes Repository geklont haben, legen Sie auf GitHub zunächst ein leeres Repository an und ändern Sie die Remote-Adresse:

```bash
git remote set-url origin https://github.com/<BENUTZERNAME>/<REPOSITORY>.git
git remote -v
```

Passen Sie außerdem `repoURL:` in `argocd-app.yaml` an dieselbe Repository-Adresse an. Speichern und übertragen Sie die Änderung:

```bash
git add argocd-app.yaml
git commit -m "ArgoCD-Repository angepasst"
git push -u origin main
```

GitHub akzeptiert bei HTTPS kein Kontopasswort mehr. Verwenden Sie einen Personal Access Token mit den erforderlichen Repository-Rechten. Schreiben Sie ihn niemals in Dateien, YAML-Manifeste oder die Remote-URL. Bei einem versehentlichen Upload muss der Token sofort widerrufen werden; das bloße Löschen der Datei reicht nicht aus, wenn der Token bereits in der Git-Historie enthalten ist.

## Sandbox stoppen und entfernen

Der folgende Befehl entfernt den gesamten k3d-Cluster. Damit werden auch die darin laufenden Anwendungen und die lokale ArgoCD-Installation entfernt:

```bash
k3d cluster delete maki-sandbox
```

Die Dateien und die Git-Historie im Repository bleiben erhalten. Zum erneuten Start führen Sie die Befehle aus dem Abschnitt **Sandbox starten** aus. Wenn ArgoCD benötigt wird, führen Sie danach auch die Schritte aus **ArgoCD installieren und verbinden** erneut aus.

## Projektstruktur

```text
maki-sandbox-gitops/
├── .gitignore
├── argocd-app.yaml
└── kubernetes/
    ├── anwendungen/
    │   └── index.html
    ├── deployment.yaml
    ├── kustomization.yaml
    ├── webserver-ingress.yaml
    └── webserver-service.yaml
```

## Fehlerbehebung

| Problem | Mögliche Ursache | Lösung |
|---|---|---|
| `Cannot connect to the Docker daemon` | Docker läuft nicht | Docker Desktop bzw. die Docker Engine starten. |
| Kubernetes-API nicht erreichbar | Cluster wurde neu erstellt oder Kubeconfig ist veraltet | `k3d cluster delete maki-sandbox` ausführen und den Cluster neu erstellen. |
| Pods in `CrashLoopBackOff` | Fehler im Image, Port oder Containerstart | `kubectl describe pod <POD-NAME>` und `kubectl logs <POD-NAME>` prüfen. |
| ArgoCD Application bleibt `OutOfSync` | Git-Änderung wurde noch nicht übernommen oder Repository ist nicht erreichbar | Repository-URL, Branch/Revision und Zugangsdaten prüfen; danach **Refresh** ausführen. |
| ArgoCD kann privates Repository nicht lesen | Repository-Zugangsdaten fehlen oder PAT ist ungültig | Unter **Settings → Repositories** die HTTPS-Verbindung mit einem gültigen PAT einrichten. |
| YAML-Fehler | Falsche Einrückung oder Tabulatoren | Nur Leerzeichen verwenden, üblicherweise zwei pro Ebene. |
| Webseite liefert `404` | Ingress, Service oder Pods sind noch nicht bereit | `kubectl get pods,services,ingress` prüfen und den Start abwarten. |

## Sicherheit und Einsatzgrenzen

Diese Sandbox ist eine lokale Lern- und Testumgebung und nicht für produktive Dienste, personenbezogene oder vertrauliche Daten bestimmt. Die Kubernetes-API wird durch die Startkonfiguration an `127.0.0.1:6443` gebunden; der Webserver ist standardmäßig nur lokal veröffentlicht. ArgoCD erhält Zugriff auf das konfigurierte Git-Repository. Verwenden Sie für private Repositories einen geeigneten Token und hinterlegen Sie ihn ausschließlich in ArgoCD, nicht im Repository.

Beachten Sie, dass ArgoCD mit `prune` Ressourcen löschen kann, die nicht mehr im Git-Zustand enthalten sind. Prüfen Sie Änderungen daher vor dem Commit und Push sorgfältig.

## Weitere Dokumentation

- [ArgoCD-Dokumentation](https://argo-cd.readthedocs.io/)
- [Kustomize-Dokumentation](https://kubectl.docs.kubernetes.io/references/kustomize/)
- [Kubernetes-Dokumentation](https://kubernetes.io/docs/)
- [k3d-Dokumentation](https://k3d.io/)
- [Docker-Dokumentation](https://docs.docker.com/)
- [Git-Dokumentation](https://git-scm.com/doc)

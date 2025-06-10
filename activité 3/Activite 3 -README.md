# Supervision des services avec Elasticsearch & Kibana (Minikube)

## 📄 Contexte de l'exercice
Dans le cadre de l'activité type 3 de l'évaluation DevOps (ECF), il était demandé de mettre en place une solution de supervision capable de collecter et visualiser les logs des services déployés. L'objectif était triple :

1. Création des fichiers
2. Déployer Elasticsearch sur un cluster Kubernetes (Minikube)
3. Déployer Kibana et le relier à Elasticsearch
4. Visualiser les logs via Kibana avec des recherches filtrées

---

## Ce qui a été réalisé
## Création fichier d'Elasticsearch sur Kubernetes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.17.16
        ports:
        - containerPort: 9200
        env:
        - name: discovery.type
          value: single-node
        - name: ES_JAVA_OPTS
          value: "-Xms512m -Xmx512m"
        volumeMounts:
        - name: esdata
          mountPath: /usr/share/elasticsearch/data
      volumes:
      - name: esdata
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: monitoring
spec:
  selector:
    app: elasticsearch
  ports:
  - port: 9200
    targetPort: 9200
    protocol: TCP
  type: ClusterIP

## Création fichier kibana.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:7.17.16
        ports:
        - containerPort: 5601
        env:
        - name: ELASTICSEARCH_HOSTS
          value: "http://elasticsearch.monitoring.svc.cluster.local:9200"
---
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: monitoring
spec:
  selector:
    app: kibana
  ports:
  - protocol: TCP
    port: 5601
    targetPort: 5601
  type: NodePort


###  Déploiement d'Elasticsearch sur Kubernetes

- Un fichier `elasticsearch.yaml` a été créé et appliqué dans le namespace `monitoring`
- Utilisation de l'image officielle d'Elastic 7.17.16
- Le service a été exposé en interne via `ClusterIP`

###  Déploiement de Kibana

- Un fichier `kibana.yaml` a été créé avec une variable d'environnement pointant vers Elasticsearch :
  ```yaml
  - name: ELASTICSEARCH_HOSTS
    value: "http://elasticsearch.monitoring.svc.cluster.local:9200"
  ```
- Exposition via un service `NodePort`, accessible avec `minikube service` ou IP directe

### 3. Visualisation des logs

- Création manuelle de logs en JSON injectés avec `curl`
- Exemple de document envoyé :
```json
{
  "timestamp": "2025-06-09T22:05:00",
  "level": "info",
  "message": "Ceci est un log pour valider l'exercice",
  "service": "api-node",
  "host": "minikube"
}
```
- Logs visibles dans Kibana via un index pattern `logs-test*` (sans champ temporel)
- Requêtes testées dans Kibana :
  - `level: "info"`
  - `message: "valider"`
  - `service: "api-node"`
  - `*` (afficher tous les documents)

---
### Envoi du log dans Elasticsearch :
curl -X POST "http://<IP_MINIKUBE>:<PORT_NODEPORT_ELASTICSEARCH>/logs-test/_doc" \
-H 'Content-Type: application/json' \
-d @test-log.json

##  Problèmes rencontrés

### 1. Kibana ne reconnaissait pas les logs injectés
**Cause :** Le champ `timestamp` n'était pas reconnu comme un champ de date valide

**Solution :** Création d'un index pattern **sans champ de date** pour permettre l'affichage brut des documents

---

##  Ce que j'ai appris (et galèrement compris)

- Comment connecter Kibana à un Elasticsearch dans Kubernetes
- Comment manipuler les services (NodePort, port-forward) pour accéder aux apps
- L'importance de bien définir les champs dans les documents JSON pour qu'ils soient exploitables dans Kibana
- Que parfois... ça ne marche pas du premier coup, mais en testant pas à pas, on y arrive !

---

## Conclusion

L'ensemble du pipeline de supervision fonctionne :
- Elasticsearch reçoit les logs
- Kibana peut les visualiser
- L'utilisateur peut filtrer par niveau, message, hôte ou service

> Mission accomplie pour cette activité type 3. Je peux maintenant expliquer comment superviser un service applicatif dans un cluster Kubernetes avec des outils open-source performants.
 j'ai réalisé sur Debian 12 via Minikube dans Oracle VirtualBox car AWS j'ai rencontré des soucis dessus
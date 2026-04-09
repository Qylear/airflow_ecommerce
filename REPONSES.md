# Réponses — TP Jour 2 : Logs E-Commerce avec HDFS

## Q1 — HDFS vs système de fichiers local

3 avantages concrets de HDFS pour 50 Go/jour de logs :

**1. Distribution et scalabilité horizontale**
Un disque local sature rapidement à 50 Go/jour. HDFS distribue les données
sur des dizaines de DataNodes — ajouter de la capacité revient à brancher
un nouveau nœud, sans interruption de service.

**2. Réplication et tolérance aux pannes**
Avec un facteur de réplication de 3, chaque bloc de log est copié sur
3 DataNodes différents. Si un serveur tombe en pleine nuit, les logs
restent accessibles depuis les 2 autres copies — aucune perte de données.

**3. Localité des données pour Spark/Hive**
Quand Spark analyse les logs, HDFS envoie le traitement là où les données
se trouvent (data locality) plutôt que de transférer 50 Go sur le réseau.
Un job Spark sur HDFS peut être 10x plus rapide qu'un équivalent sur NFS
grâce à ce principe.

---

## Q2 — NameNode, point de défaillance unique (SPOF)

**Si le NameNode tombe :**
- Les DataNodes continuent à tourner et conservent leurs blocs
- Mais aucun client ne peut lire ni écrire : le NameNode est le seul
  à connaître la localisation des blocs (namespace)
- Le cluster HDFS est totalement inaccessible jusqu'au redémarrage

**Solution : HDFS NameNode HA (High Availability)**
Hadoop propose un mode actif/passif avec deux NameNodes :
- NameNode Actif : répond aux requêtes clients
- NameNode Standby : synchronisé en temps réel, prêt à prendre le relais

**Rôle du JournalNode :**
Les JournalNodes forment un quorum (minimum 3) qui stocke le journal
des modifications du namespace (EditLog). Le NameNode Actif écrit chaque
opération dans ce journal, le Standby le lit en continu. En cas de panne,
le Standby a une vue identique du namespace et peut devenir Actif en
quelques secondes (failover automatique via ZooKeeper).

---

## Q3 — HdfsSensor : mode poke vs reschedule

| | poke | reschedule |
|---|---|---|
| Comportement | Occupe le worker en continu | Libère le worker entre chaque poke |
| Consommation worker | 1 slot bloqué pendant toute l'attente | 0 slot entre les pokes |
| Usage recommandé | Attentes courtes (< 1 min) | Attentes longues (plusieurs minutes) |

**Scénario de blocage avec poke + CeleryExecutor :**
Si on a 4 workers Celery et 4 pipelines parallèles, chacun avec un
HdfsSensor en mode poke attendant un fichier qui n'arrive qu'en 1h,
les 4 workers sont bloqués pendant 1h — aucune autre tâche du cluster
ne peut s'exécuter. Avec reschedule, les workers sont libérés entre
chaque vérification et peuvent traiter d'autres tâches.

En production : toujours reschedule pour les sensors avec poke_interval > 30s.

---

## Q4 — Réplication HDFS et cohérence

**Écriture d'un bloc de 128 Mo avec réplication=3 :**
1. Le client demande au NameNode où écrire → reçoit 3 adresses DataNode
2. Le client ouvre un pipeline : DN1 → DN2 → DN3
3. Le client envoie les données à DN1, qui les forward à DN2, qui les
   forward à DN3 (pipeline séquentiel, pas 3 uploads parallèles)
4. Chaque DN envoie un ACK vers l'amont quand son écriture est terminée
5. Le client reçoit l'ACK final quand les 3 copies sont écrites
→ 3 copies, sur 3 DataNodes distincts, écriture en pipeline

**Cohérence lors d'une lecture concurrente :**
HDFS utilise un modèle write-once, read-many : un fichier en cours
d'écriture n'est pas visible pour les lecteurs tant qu'il n'est pas
fermé (close). Un lecteur ne peut pas lire un bloc partiellement écrit.
C'est pourquoi le HdfsSensor est utile : il attend que le fichier soit
complet et visible avant de lancer l'analyse.

---

## Captures d'écran

_À compléter après exécution :_
1. `docker compose ps` — tous les conteneurs running/healthy
2. `docker exec namenode hdfs dfsadmin -report` — 1 DataNode Live
3. HDFS Web UI (http://localhost:9870) — Browse Directory /data/ecommerce/logs/raw/
4. Airflow UI — Vue Graph du DAG logs_ecommerce_dag (8 tâches)
5. Airflow UI — Exécution complète avec les 2 branches (une verte, une grisée)
6. Logs de analyser_logs_hdfs — status codes et Top 5 URLs
7. HDFS Web UI — fichier dans /data/ecommerce/logs/processed/

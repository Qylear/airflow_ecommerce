Q1 — HDFS vs système de fichiers local Pourquoi ne pas simplement stocker les logs sur le disque
local du serveur Airflow ou sur un NFS ? Listez 3 avantages concrets de HDFS pour un cas d’usage de
50 Go/jour de logs, en vous appuyant sur les caractéristiques du système (distribution, réplication,
localité des données).

si c'est en local et qu'on pert le serveur airflow ont pert tout.

Plusieurs workers peuvent écrire/lire sans conflit
Les logs restent disponibles même si un nœud tombe
Conçu pour gérer énormes volumes de logs


Q2 — NameNode, point de défaillance unique (SPOF) Dans l’architecture HDFS, le NameNode est
un SPOF. Si le NameNode tombe, que se passe-t-il pour les DataNodes et pour les clients ? Quels
mécanismes Hadoop propose-t-il pour pallier ce problème en production (HDFS NameNode HA) ?
Quel est le rôle du Journal Node dans cette architecture haute disponibilité ?

côté Datanodes : il continue de tourner, il garde les données en local mais il ne sais plus quoi en faire 
côté client : Impossibilité de lire les fichier de log, écrire sur les fichier et de lister les dossiers

les solutions apportés est le NameNodes HA qui est composé de plusieurs choses :
  - Active Namenode
  - Standby Namenode
  - JournalNodes

Le JournalNode synchronise les métadonnées entre les NameNodes, il garantit la cohérence et la reprise sans perte


Q3 — HdfsSensor vs polling actif Comparez le HdfsSensor en mode poke et en mode reschedule .
Dans quel cas utiliseriez-vous l’un plutôt que l’autre ? Quel est l’impact sur le nombre de
slots de workers Airflow disponibles ?
Proposez un scénario concret où le mauvais choix de mode bloquerait tout le scheduler.

| | poke | reschedule |
|---|---|---|
| Comportement | Occupe le worker en continu | Libère le worker entre chaque poke |
| Consommation worker | 1 slot bloqué pendant toute l'attente | 0 slot entre les pokes |
| Usage recommandé | Attentes courtes (< 1 min) | Attentes longues (plusieurs minutes) |

Si on a 4 workers Celery et 4 pipelines parallèles, chacun avec un
HdfsSensor en mode poke attendant un fichier qui n'arrive qu'en 1h,
les 4 workers sont bloqués pendant 1h — aucune autre tâche du cluster
ne peut s'exécuter. Avec reschedule, les workers sont libérés entre
chaque vérification et peuvent traiter d'autres tâches.

En production : toujours reschedule pour les sensors avec poke_interval > 30s


Q4 — Réplication HDFS et cohérence des données Dans le 
hadoop.env , on a
HDFS_CONF_dfs_replication=1 (réplication minimale). En production avec un facteur de réplication
de 3, expliquez ce qui se passe lors de l’écriture d’un bloc de 128 Mo : combien de copies sont écrites,
sur combien de DataNodes, et dans quel ordre ? Que garantit HDFS en termes de cohérence lors
d’une lecture concurrente (pendant que l’écriture est en cours) ?

il sera répliqué 3 fois sur 3Datanodes différents et façon enchainé (Client → DN1 → DN2 → DN3) 

pas de données corrompues visibles, et uniquement en lecture


1. `docker compose ps` — tous les conteneurs running/healthy
   <img width="1910" height="138" alt="image" src="https://github.com/user-attachments/assets/70f06d5c-9a64-46dc-a8d7-59105c648dfa" />

2. `docker exec namenode hdfs dfsadmin -report` — 1 DataNode Live

   <img width="833" height="720" alt="image" src="https://github.com/user-attachments/assets/dc40780e-2f32-40e9-9ddd-24ade9b60029" />

3. HDFS Web UI (http://localhost:9870) — Browse Directory /data/ecommerce/logs/raw/

   <img width="1185" height="313" alt="image" src="https://github.com/user-attachments/assets/0e3b6a08-fa03-477a-b149-f0c237bb9c4d" />

4. Airflow UI — Vue Graph du DAG logs_ecommerce_dag (8 tâches)

   <img width="1541" height="198" alt="image" src="https://github.com/user-attachments/assets/6074204c-1b7c-4dde-8c55-01ca1510ba3e" />

5. Airflow UI — Exécution complète avec les 2 branches (une verte, une grisée)
 
    <img width="474" height="175" alt="image" src="https://github.com/user-attachments/assets/197a23be-ece7-428d-9db7-7b911a9b1cb7" />
    
6. Logs de analyser_logs_hdfs — status codes et Top 5 URLs

    <img width="626" height="197" alt="image" src="https://github.com/user-attachments/assets/8d775953-1b6b-4e2d-a96a-b9ce2f328bc9" />

7. HDFS Web UI — fichier dans /data/ecommerce/logs/processed/

  <img width="1247" height="376" alt="image" src="https://github.com/user-attachments/assets/cb63e663-a14c-4dd8-91d3-519eaf2a8743" />








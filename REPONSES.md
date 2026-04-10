Q1 — HDFS vs système de fichiers local Pourquoi ne pas simplement stocker les logs sur le disque
local du serveur Airflow ou sur un NFS ? Listez 3 avantages concrets de HDFS pour un cas d’usage de
50 Go/jour de logs, en vous appuyant sur les caractéristiques du système (distribution, réplication,
localité des données).





Q2 — NameNode, point de défaillance unique (SPOF) Dans l’architecture HDFS, le NameNode est
un SPOF. Si le NameNode tombe, que se passe-t-il pour les DataNodes et pour les clients ? Quels
mécanismes Hadoop propose-t-il pour pallier ce problème en production (HDFS NameNode HA) ?
Quel est le rôle du Journal Node dans cette architecture haute disponibilité ?






Q3 — HdfsSensor vs polling actif Comparez le 
HdfsSensor en mode 
poke et en mode
reschedule . Dans quel cas utiliseriez-vous l’un plutôt que l’autre ? Quel est l’impact sur le nombre de
slots de workers Airflow disponibles ? Proposez un scénario concret où le mauvais choix de mode
bloquerait tout le scheduler.






Q4 — Réplication HDFS et cohérence des données Dans le 
hadoop.env , on a
HDFS_CONF_dfs_replication=1 (réplication minimale). En production avec un facteur de réplication
de 3, expliquez ce qui se passe lors de l’écriture d’un bloc de 128 Mo : combien de copies sont écrites,
sur combien de DataNodes, et dans quel ordre ? Que garantit HDFS en termes de cohérence lors
d’une lecture concurrente (pendant que l’écriture est en cours) ?




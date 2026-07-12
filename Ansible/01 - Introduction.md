# Qu'est ce que Ansible ?

Ansible est un outil open source largement utilisé en DevOps pour automatiser 
la configuration et la gestion de machines distantes, aussi bien sous Linux que sous Windows. 
Il a été créé en 2012 par Michael DeHaan.

L’une des principales particularités d’Ansible, par rapport à des outils comme Puppet ou Chef, est son fonctionnement “agentless”. 
Cela signifie qu’aucun agent spécifique n’a besoin d’être installé sur les machines cibles. 
Ansible est donc léger et rapide à déployer. Il suffit que le service SSH soit accessible depuis le serveur 
Ansible pour les machines Linux. Pour les systèmes Windows, Ansible utilise le protocole WinRM, 
qui repose sur des communications sécurisées via SSL, à l’image d’un serveur HTTPS. Les machines distantes 
doivent également disposer de Python (côté Linux), car Ansible s’appuie majoritairement sur ce langage pour exécuter ses modules.

Pour appliquer les configurations, Ansible se connecte à chaque machine cible via SSH (ou WinRM pour Windows) 
et exécute les tâches définies. Lorsqu’un grand nombre de machines est concerné, les opérations sont exécutées en parallèle, 
ce qui permet un gain de temps significatif. Par défaut, Ansible utilise une stratégie d’exécution où chaque tâche 
doit être terminée sur l’ensemble des hôtes avant de passer à la suivante.

Ansible repose sur le principe d’idempotence. Cela signifie qu’une tâche peut être exécutée plusieurs fois 
sans modifier le résultat final si l’état souhaité est déjà atteint. Par exemple, lors de la gestion d’un utilisateur, 
Ansible vérifiera son existence : s’il est déjà présent, aucune action ne sera effectuée ; sinon, il sera créé.

Contrairement à Ansible, des outils comme Puppet fonctionnent selon un modèle “pull”. 
Chaque machine dispose d’un agent qui interroge un serveur central (Puppet Master) pour récupérer sa configuration, 
définie sous forme de classes et de ressources. Ansible adopte principalement un modèle “push”, où le serveur envoie 
directement les configurations vers les machines cibles. Il peut toutefois effectuer certaines actions de type “pull”, 
comme récupérer des fichiers depuis un hôte distant pour les redistribuer ailleurs.

# Architecture Ansible

D’un point de vue logique, une infrastructure Ansible s’articule autour de deux éléments principaux :

* Le nœud de contrôle (control node) : il s’agit de la machine sur laquelle Ansible est installé. Ce serveur pilote l’ensemble des opérations d’automatisation via les commandes ansible et ansible-playbook. Il doit être basé sur un système Linux et disposer d’un accès réseau (SSH ou WinRM) vers les machines cibles.

* Les nœuds gérés (managed nodes ou hosts) : ce sont les machines sur lesquelles Ansible exécute les tâches d’automatisation. Aucun agent Ansible n’y est installé. Elles doivent simplement être accessibles depuis le nœud de contrôle et correctement configurées (accès SSH ou WinRM, Python pour Linux).
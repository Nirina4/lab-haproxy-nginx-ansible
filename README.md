# Lab HAProxy + Nginx

## Groupe

- RAKOTONIRINA Fanambinantsoa Heriniaina Josoa
- EMADISSON Aina Horavaka
- RAHARINJATOVO Antsa Fanomezantsoa Romance

Projet de notre module DevOps. Le but c'etait de monter un reverse proxy
HAProxy devant plusieurs serveurs Nginx, avec du HTTPS et de la
repartition de charge, et automatiser tout ca avec Ansible.

Architecture en gros : le client tape sur HAProxy en HTTPS (port 443),
HAProxy dechiffre et envoie la requete vers un des 6 Nginx en HTTP
derriere, selon un round-robin classique.

## Comment ca marche

On a d'abord teste chaque brique a la main avant d'automatiser quoi que
ce soit : le reseau Docker (verifier que deux conteneurs se parlent par
leur nom), un Nginx tout seul, le certificat SSL, HAProxy avec un seul
backend, puis 2, puis 6. Une fois que chaque etape etait comprise on a
ecrit les taches Ansible correspondantes.

Les 4 roles Ansible :
- network : cree le reseau docker
- nginx : genere la conf + la page html de chaque serveur et lance les
  6 conteneurs
- certs : genere le certificat SSL et le combine en pem
- haproxy : genere haproxy.cfg (avec les 6 backends generes via une
  boucle jinja2) et lance le conteneur

## Lancer le projet

Il faut Docker, Ansible, la collection community.docker et le module
python docker installes avant.

```bash
git clone https://github.com/Nirina4/lab-haproxy-nginx-ansible.git
cd lab-haproxy-nginx-ansible
ansible-playbook site.yml
```

Ensuite `docker ps` doit montrer 7 conteneurs (haproxy_lb + 6 nginx), et
`curl -vk https://localhost/` doit repondre en 200.

## Un bug qu'on a eu

Le certificat pem genere par openssl avait des permissions trop
restrictives (0600), du coup HAProxy n'arrivait pas a le lire dans le
conteneur et plantait au demarrage. On a du ajouter une tache dans le
role certs pour repasser le pem en 0644 apres generation.

Deuxieme truc : le port 443 est un port protege, il a fallu ajouter la
capability NET_BIND_SERVICE au conteneur haproxy pour qu'il puisse s'y
attacher sans etre root.

## Tests

- acces https qui repond bien avec le certificat
- round robin verifie en tapant plusieurs fois sur https://localhost/
- coupe un conteneur nginx (docker stop) et verifie que les requetes
  continuent d'arriver sur les autres
- page de stats haproxy sur le port 8404/stats pour voir l'etat des
  backends

Ce fichier documente les étapes pour déployer deux instances WordPress (wordpress1 et wordpress2) derrière un serveur Nginx configuré en tant que reverse proxy avec redirection HTTPS. Il utilise Docker pour la gestion des services.
Prérequis

    Serveur avec Docker et Docker Compose installés.
    Nginx configuré avec les certificats SSL (Let's Encrypt ou auto-signés).
    Domaines ou sous-domaines configurés pour pointer vers votre serveur (par exemple jhennebo.be).
    
- Assurez-vous qu'aucune autre instance de Nginx ne tourne sur le serveur. Vous pouvez vérifier cela avec la commande suivante :
  ```bash
  sudo systemctl status nginx

Si Nginx est actif, vous devez l'arrêter avant de commencer :

bash

sudo systemctl stop nginx
Structure du projet

.
├── conf.d
├── html
├── mariadb_data
│   ├── mysql
│   ├── performance_schema
│   ├── sys
│   ├── wordpress1
│   └── wordpress2
├── nginx
│   ├── conf.d
│   ├── default.conf
│   ├── logs
│   └── ssl
└── wordpress
    ├── wp1
    └── wp2

Étapes d'installation
1. Créer le fichier docker-compose.yaml

Le fichier docker-compose.yaml permet de gérer les conteneurs WordPress, MariaDB et Nginx.

yaml

version: '3'

services:
  db:
    image: mariadb:latest
    container_name: db
    environment:
      MYSQL_ROOT_PASSWORD: my_root_password
    volumes:
      - ./mariadb_data:/var/lib/mysql
    user: "www-data"
    networks:
      - wp-network

  wordpress1:
    image: wordpress:latest
    container_name: wordpress1
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wp1
      WORDPRESS_DB_PASSWORD: wordpress1234
      WORDPRESS_DB_NAME: wordpress1
    volumes:
      - ./wordpress/wp1:/var/www/html
    depends_on:
      - db
    networks:
      - wp-network

  wordpress2:
    image: wordpress:latest
    container_name: wordpress2
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wp2
      WORDPRESS_DB_PASSWORD: wordpress1234
      WORDPRESS_DB_NAME: wordpress2
    volumes:
      - ./wordpress/wp2:/var/www/html
    depends_on:
      - db
    networks:
      - wp-network

  nginxrp:
    image: nginx:latest
    container_name: nginxrp
    ports:
      - "443:443"
      - "8080:80"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./nginx/ssl:/etc/nginx/ssl
      - ./html:/usr/share/nginx/html
    depends_on:
      - wordpress1
      - wordpress2
    networks:
      - wp-network

networks:
  wp-network:
    driver: bridge

2. Configuration Nginx

Créer un fichier de configuration pour Nginx dans nginx/conf.d/test.conf:

nginx

# Redirection HTTP vers HTTPS
server {
    listen 80;
    server_name jhennebo.be www.jhennebo.be;

    # Redirection vers HTTPS
    return 301 https://$host$request_uri;
}

# Configuration pour HTTPS et Reverse Proxy
server {
    listen 443 ssl;
    server_name jhennebo.be www.jhennebo.be;

    # Certificats SSL
    ssl_certificate /etc/nginx/ssl/full_chain.pem;
    ssl_certificate_key /etc/nginx/ssl/jhennebo.be_private_key.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Proxy pour wordpress1
    location /wordpress1/ {
        proxy_pass http://wordpress1:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Proxy pour wordpress2
    location /wordpress2/ {
        proxy_pass http://wordpress2:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Serveur web statique par défaut (pour tester Nginx)
    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
}

3. Générer les certificats SSL
créer le fichier full_chain.pem en utilisant les certificats telechargés. Étant donné que tu as plusieurs fichiers, tu vas combiner le certificat principal et les certificats intermédiaires dans le bon ordre.
Étapes pour Créer full_chain.pem

    Vérifier l’Ordre des Certificats : Assure-toi d'avoir les fichiers suivants :
        Certificat principal : jhennebo.be_ssl_certificate.cer
        Certificats intermédiaires : intermediate1.cer et intermediate2.cer
        Clé privée : jhennebo.be_private_key.key (non incluse dans full_chain.pem, mais nécessaire pour la configuration Nginx)

    Créer full_chain.pem : Utilise la commande cat pour concaténer le certificat principal avec les certificats intermédiaires. Voici comment faire :

    bash

cat jhennebo.be_ssl_certificate.cer intermediate1.cer intermediate2.cer >
Placer les certificats SSL dans nginx/ssl/ :

    full_chain.pem : Chaîne complète du certificat
    jhennebo.be_private_key.key : Clé privée associée au certificat


4. Gestion des Permissions dans Docker
    4.1. Permissions sur l'Hôte

    Assure-toi que les fichiers et répertoires que tu souhaites partager avec tes conteneurs Docker ont les bonnes permissions. Voici quelques commandes utiles :

    Vérifier les permissions d'un répertoire :

    bash

    ls -l /chemin/vers/ton/répertoire

   Changer le propriétaire d'un répertoire (par exemple, pour donner la propriété à l'utilisateur www-data qui est souvent utilisé par Nginx et PHP) :

   bash

   sudo chown -R www-data:www-data /chemin/vers/ton/répertoire

   Modifier les permissions d'un répertoire (par exemple, pour donner des droits de lecture, écriture et exécution à l'utilisateur et aux groupes) :

   bash

  sudo chmod -R 755 /chemin/vers/ton/répertoire

  Pour des fichiers spécifiques, tu peux faire :

  bash

    sudo chmod 644 /chemin/vers/ton/fichier

  4.2 Permissions à l'Intérieur des Conteneurs

  Lorsque tu exécutes des conteneurs Docker, il est souvent nécessaire de s'assurer que les permissions sont correctes à l'intérieur du conteneur, notamment pour les applications web comme WordPress.

    Accéder à un conteneur en cours d'exécution :

    bash

   docker exec -it nom_du_conteneur bash

  Vérifier les permissions d'un répertoire à l'intérieur d'un conteneur :

  bash

  ls -l /chemin/vers/ton/répertoire

  Changer le propriétaire d'un répertoire à l'intérieur d'un conteneur (par exemple, pour un conteneur WordPress) :

  bash

  chown -R www-data:www-data /var/www/html

  Modifier les permissions d'un répertoire à l'intérieur d'un conteneur :

  bash

    chmod -R 755 /var/www/html

  4.3. Utilisation de Docker Compose

  Si tu utilises Docker Compose, tu peux définir les permissions directement dans ton fichier docker-compose.yaml :

  yaml

  services:
    wordpress:
      image: wordpress
      volumes:
        - ./wordpress/wp1:/var/www/html
      user: "www-data"  # Exécute le conteneur en tant qu'utilisateur www-data

   Cela permet de s'assurer que le conteneur WordPress s'exécute avec les permissions appropriées.
    
   4.4 Vérification des Permissions après Redémarrage

    Après avoir modifié les permissions ou redémarré les conteneurs, assure-toi de vérifier que tout fonctionne comme prévu. Tu peux le faire en consultant les logs des conteneurs :

    bash

    docker-compose logs





5. Démarrer les conteneurs

Dans le répertoire racine du projet, exécuter la commande suivante pour démarrer tous les services :

bash

docker-compose up -d

6. Configuration des bases de données WordPress

Se connecter à MariaDB pour créer les bases de données et utilisateurs pour les deux WordPress :

bash

docker exec -it db mysql -u root -p

# Créer la base de données et l'utilisateur pour wordpress1
CREATE DATABASE wordpress1;
CREATE USER 'wp1'@'%' IDENTIFIED BY 'wordpress1234';
GRANT ALL PRIVILEGES ON wordpress1.* TO 'wp1'@'%';

# Créer la base de données et l'utilisateur pour wordpress2
CREATE DATABASE wordpress2;
CREATE USER 'wp2'@'%' IDENTIFIED BY 'wordpress1234';
GRANT ALL PRIVILEGES ON wordpress2.* TO 'wp2'@'%';

7. Installer WordPress

    Accéder à https://jhennebo.be/wordpress1/wp-admin/install.php et suivre les instructions pour installer WordPress.
    Répéter pour https://jhennebo.be/wordpress2/wp-admin/install.php.

8. Ajuster les URLs WordPress (si nécessaire)

Vérifier les URL dans la base de données pour chaque instance WordPress et s'assurer qu'elles pointent vers les bons chemins :

bash

# Se connecter à MariaDB pour wordpress1
docker exec -it db mysql -u root -p

USE wordpress1;
SELECT option_name, option_value FROM wp_options WHERE option_name IN ('siteurl', 'home');

# Si nécessaire, les corriger
UPDATE wp_options SET option_value = 'https://jhennebo.be/wordpress1' WHERE option_name = 'siteurl' OR option_name = 'home';



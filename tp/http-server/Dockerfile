# Image de base httpd
FROM httpd:2.4

# On rajoute notre configuration perso dans le container
COPY httpd.conf /usr/local/apache2/conf/

# On copie nos fichiers dans le container
COPY index.html /usr/local/apache2/htdocs/

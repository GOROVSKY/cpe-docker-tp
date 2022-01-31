# Image de base
FROM postgres:11.6-alpine

# Ajout des scripts sql à exucter au run
ADD 01-CreateScheme.sql /docker-entrypoint-initdb.d/01-CreateScheme.sql
ADD 02-InsertData.sql /docker-entrypoint-initdb.d/02-InsertData.sql

# Variables d'environnement
ENV POSTGRES_DB=db \
POSTGRES_USER=usr \
POSTGRES_PASSWORD=pwd
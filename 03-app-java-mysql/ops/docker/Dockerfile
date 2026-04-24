# Utilisation d'une image de base Java optimisée
FROM openjdk:17-jdk-slim

# Définir le répertoire de travail à l'intérieur du conteneur
WORKDIR /app

# Copier le fichier JAR dans le conteneur
# /!\ adapater le nom de chemin et/ou fichier
COPY target/spring-petclinic-3.4.0-SNAPSHOT.jar app.jar

# Exposer le port sur lequel l'application s'exécute
EXPOSE 8080

# Commande pour exécuter l'application
ENTRYPOINT ["java", "-jar", "app.jar"]

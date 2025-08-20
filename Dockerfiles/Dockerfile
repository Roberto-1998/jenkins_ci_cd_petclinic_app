# Imagen base con JDK para ejecutar el JAR
FROM eclipse-temurin:21-jdk-jammy

# Directorio de trabajo en el contenedor
WORKDIR /app

# Copiamos el artefacto descargado por Jenkins desde Nexus
COPY petclinic-app-*.jar app.jar

# Exponemos el puerto en el que corre la app (Spring Boot usa 8080 por defecto)
EXPOSE 8080

# Comando para ejecutar la aplicación
ENTRYPOINT ["java", "-jar", "app.jar"]
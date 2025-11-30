# Stage 1: Build the application
FROM maven:3.9.6-eclipse-temurin-17-alpine AS build

WORKDIR /app

# Copy only the necessary files first (speeds up caching)
COPY pom.xml .
RUN mvn dependency:go-offline

# Copy the rest of the source code
COPY src ./src

# Build the Spring Boot jar
RUN mvn clean package -DskipTests

# Stage 2: Run the application
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

# Copy the jar from the build stage
COPY --from=build /app/target/*.jar app.jar

# Expose port (change if needed)
EXPOSE 8080

# Run the app
ENTRYPOINT ["java","-jar","app.jar"]


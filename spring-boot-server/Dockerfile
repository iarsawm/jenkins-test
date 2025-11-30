### Multi-stage Dockerfile for Spring Boot backend
#  - Stage 1: build with Maven
#  - Stage 2: run with a slim JRE

FROM maven:3.9.5-openjdk-17-slim AS build
WORKDIR /build
# copy maven wrapper and pom first for better caching
COPY pom.xml mvnw .mvn/ ./
COPY .mvn .mvn
# copy the rest of the source
COPY src ./src
RUN mvn -B -DskipTests package

FROM eclipse-temurin:17-jre-jammy
WORKDIR /app
ARG JAR_FILE=/build/target/*.jar
COPY --from=build ${JAR_FILE} /app/app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]

# ---- Stage 1: build the jar with Maven + JDK 25 ----
FROM maven:3.9-eclipse-temurin-25 AS build
WORKDIR /src

# Download dependencies first so this layer is cached when only code changes
COPY pom.xml .
RUN mvn -B -q dependency:go-offline

COPY src ./src
RUN mvn -B -q package -DskipTests \
 && cp target/*.jar /src/app.jar

# ---- Stage 2: small runtime image with only the JRE ----
FROM eclipse-temurin:25-jre
WORKDIR /app

# Run as a non-root user
RUN useradd --system --uid 10001 spring
USER 10001

COPY --from=build /src/app.jar app.jar

EXPOSE 8080
# Size the JVM heap from the container memory limit
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "-jar", "/app/app.jar"]

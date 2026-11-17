# spring-boot-data-mongodb

The REST API for the **Tutorials** app. It's a Spring Boot service that stores tutorials in MongoDB and serves them as JSON under `/api/tutorials`. The Angular frontend in [../angular-crud](../angular-crud) uses it.

## Tech stack

| | |
|---|---|
| Language | Java 25 |
| Framework | Spring Boot 3.5.16 (Spring Web, Spring Data MongoDB, Actuator) |
| Database | MongoDB (7 in Docker and Kubernetes), database `belajar_db`, collection `tutorials` |
| Build | Maven, through the included wrapper (`mvnw`) |

## Project structure

```
src/main/java/com/belajar/spring/data/mongodb/
├── SpringBootDataMongodbApplication.java   # entry point
├── controller/TutorialController.java      # REST endpoints under /api
├── model/Tutorial.java                     # @Document(collection = "tutorials")
└── repository/TutorialRepository.java      # MongoRepository + findByPublished, findByTitleContaining
src/main/resources/application.properties   # local defaults and actuator settings
```

## Data model

```json
{
  "id": "6afc0fec98909a5f141affb1",
  "title": "Spring Boot",
  "description": "Getting started",
  "published": false
}
```

MongoDB generates the `id`. New tutorials always start with `published: false`.

## API

| Method | Path | Body | Success | Notes |
|---|---|---|---|---|
| GET | `/api/tutorials` | | 200 with a list | **204** (no body) when there are none |
| GET | `/api/tutorials?title=abc` | | 200 with a list | Title contains `abc` (case-sensitive); 204 when nothing matches |
| GET | `/api/tutorials/published` | | 200 with a list | 204 when there are none |
| GET | `/api/tutorials/{id}` | | 200 with one tutorial | 404 if not found |
| POST | `/api/tutorials` | `{"title", "description"}` | 201 with the created tutorial | |
| PUT | `/api/tutorials/{id}` | `{"title", "description", "published"}` | 200 with the updated tutorial | 404 if not found |
| DELETE | `/api/tutorials/{id}` | | 204 | Also 204 if the id doesn't exist |
| DELETE | `/api/tutorials` | | 204 | **Deletes every tutorial** |

There's no authentication: anyone who can reach the API can change or delete data.

Try it with curl:

```bash
curl -i http://localhost:8080/api/tutorials
curl -i -X POST http://localhost:8080/api/tutorials \
     -H "Content-Type: application/json" \
     -d '{"title":"Spring Boot","description":"Getting started"}'
```

### Health endpoints

| Path | Used for |
|---|---|
| `/actuator/health` | overall status |
| `/actuator/health/liveness` | Kubernetes liveness and startup probes |
| `/actuator/health/readiness` | Kubernetes readiness probe |

## Configuration

Settings come from [application.properties](src/main/resources/application.properties). Any of them can be overridden by an environment variable: Spring Boot maps `SPRING_DATA_MONGODB_HOST` to `spring.data.mongodb.host`, and so on.

| Property | Environment variable | Local default | In Kubernetes |
|---|---|---|---|
| `spring.data.mongodb.host` | `SPRING_DATA_MONGODB_HOST` | `localhost` | `mongodb` |
| `spring.data.mongodb.port` | `SPRING_DATA_MONGODB_PORT` | `27017` | `27017` |
| `spring.data.mongodb.database` | `SPRING_DATA_MONGODB_DATABASE` | `belajar_db` | `belajar_db` |
| `spring.data.mongodb.username` | `SPRING_DATA_MONGODB_USERNAME` | none | from the Secret |
| `spring.data.mongodb.password` | `SPRING_DATA_MONGODB_PASSWORD` | none | from the Secret |
| `spring.data.mongodb.authentication-database` | `SPRING_DATA_MONGODB_AUTHENTICATION_DATABASE` | none | `admin` |

The Kubernetes values are set in [../k8s/backend/configmap.yaml](../k8s/backend/configmap.yaml) and [../k8s/mongodb/secret.yaml](../k8s/mongodb/secret.yaml).

The server listens on port **8080**.

## Run locally

Requirements: **JDK 25** and Docker (for MongoDB). Maven is downloaded by the wrapper.

1. Start MongoDB:
   ```bash
   docker run -d --name belajar-mongo -p 27017:27017 mongo:7    # first time
   docker start belajar-mongo                                   # after that
   ```
2. Point `JAVA_HOME` at JDK 25 if it isn't already:
   ```powershell
   $env:JAVA_HOME = "C:\Program Files\Java\jdk-25.0.4"          # PowerShell
   ```
   ```bash
   export JAVA_HOME="/c/Program Files/Java/jdk-25.0.4"          # Git Bash
   ```
3. Start the app:
   ```bash
   ./mvnw spring-boot:run        # PowerShell: .\mvnw spring-boot:run
   ```
4. Check it: http://localhost:8080/api/tutorials should return 204 (empty) or a list.

To use the UI as well, start the frontend afterwards. See [../angular-crud](../angular-crud).

## Build and test

```bash
./mvnw clean package             # runs the tests and builds target/spring-boot-data-mongodb-0.0.1-SNAPSHOT.jar
./mvnw test                      # tests only
```

There's one test, `contextLoads`, which checks that the application starts. It doesn't need a running MongoDB.

If requests suddenly fail with `Unresolved compilation problem`, your IDE has written broken classes into `target/`. Run `./mvnw clean compile` and restart.

## Docker

The [Dockerfile](Dockerfile) has two stages: Maven with JDK 25 builds the jar, then a JRE 25 image runs it as a non-root user, with the heap sized from the container's memory limit (`-XX:MaxRAMPercentage=75`).

```bash
docker build -t belajar-backend:1.0.0 .
```

For running it in Kubernetes, see [../k8s/README.md](../k8s/README.md).

## Notes

- The controller still has `@CrossOrigin(origins = "http://localhost:8081")`. It isn't needed any more: the frontend calls the API on the same address (through the `ng serve` proxy locally, or the Ingress in Kubernetes), so the browser never makes a cross-origin request.
- Errors inside the endpoints are returned as a 500 with an empty body and aren't logged.

# angular-crud

The web frontend for the **Tutorials** app: an Angular 17 single-page app for creating, listing, searching, editing, publishing and deleting tutorials. It talks to the Spring Boot REST API in [../spring-boot-data-mongodb](../spring-boot-data-mongodb), which stores the data in MongoDB.

## Features

- List all tutorials and select one to see its details
- Search tutorials by title
- Add a tutorial (title and description)
- Edit title and description, publish or unpublish, delete
- Delete all tutorials

Each tutorial has `id`, `title`, `description` and `published`.

## Tech stack

| | |
|---|---|
| Framework | Angular 17 (NgModules, with the new `@if` / `@for` template syntax) |
| Styling | Bootstrap 4.6 |
| HTTP | `HttpClient`, through one service: [tutorial.service.ts](src/app/services/tutorial.service.ts) |
| Build | Angular CLI, `application` builder (esbuild) |
| Production server | nginx (see [Dockerfile](Dockerfile)) |

## Project structure

```
src/app/
├── app.module.ts                 # declares the components, imports Forms/Http/Router
├── app-routing.module.ts         # routes (below)
├── app.component.*               # navbar + <router-outlet>
├── models/tutorial.model.ts      # Tutorial type
├── services/tutorial.service.ts  # all API calls
└── components/
    ├── tutorials-list/           # list, search, "Remove All", shows details of the selected item
    ├── tutorial-details/         # view mode (in the list) and edit mode (/tutorials/:id)
    └── add-tutorial/             # create form
```

| Route | Page |
|---|---|
| `/` | redirects to `/tutorials` |
| `/tutorials` | list and search |
| `/tutorials/:id` | edit one tutorial |
| `/add` | add a tutorial |

## How it calls the API

The app calls the **relative** URL `/api/tutorials`, never a hardcoded host. Whatever server delivered the page also answers the API calls, so the browser sees a single origin and CORS never comes into play.

- **Local development:** `ng serve` forwards `/api` to `http://localhost:8080` using [proxy.conf.json](proxy.conf.json).
- **Kubernetes:** the Traefik Ingress routes `/api` to the backend and everything else to this app. See [../k8s/README.md](../k8s/README.md).

| Method | Path | Used for |
|---|---|---|
| GET | `/api/tutorials` | list all |
| GET | `/api/tutorials?title=...` | search by title |
| GET | `/api/tutorials/{id}` | load one |
| POST | `/api/tutorials` | create |
| PUT | `/api/tutorials/{id}` | update, publish or unpublish |
| DELETE | `/api/tutorials/{id}` | delete one |
| DELETE | `/api/tutorials` | delete all |

## Run locally

Requirements: Node.js 18 or 20 (the versions Angular 17 supports) and npm.

1. Start MongoDB and the backend first, as described in [../spring-boot-data-mongodb](../spring-boot-data-mongodb). For example:
   ```bash
   docker start belajar-mongo        # or: docker run -d --name belajar-mongo -p 27017:27017 mongo:7
   cd ../spring-boot-data-mongodb && ./mvnw spring-boot:run
   ```
2. Install dependencies (first time only) and start the dev server:
   ```bash
   npm install
   npm start                         # same as: ng serve (port 8081 and the /api proxy are set in angular.json)
   ```
3. Open **http://localhost:8081**. The page reloads automatically when you change a source file.

If the page loads but the list stays empty with errors in the browser console, the backend isn't running on port 8080.

## Build

```bash
npm run build                        # production build into dist/angular-17-crud/browser
```

## Docker

The [Dockerfile](Dockerfile) has two stages: Node 20 runs `npm ci` and the production build, then nginx serves the result. Paths like `/tutorials/42` fall back to `index.html` so the Angular router can handle them.

```bash
docker build -t belajar-frontend:1.0.0 .
```

In Kubernetes, the nginx config comes from the ConfigMap in [../k8s/frontend/configmap.yaml](../k8s/frontend/configmap.yaml) instead of the one built into the image.

## Tests

```bash
npm test                             # Karma + Jasmine
```

The spec files are the ones the Angular CLI generated and haven't been updated. They don't set up `HttpClient` or the router for testing, so expect them to fail until they're fixed.

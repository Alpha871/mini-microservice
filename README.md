# Blog - Microservices Architecture

A microservices-based blog application built with **Node.js**, **React**, and **Kubernetes**. This project demonstrates key distributed systems patterns including **Event-Driven Architecture**, **CQRS**, and **Event Sourcing**.

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                            NGINX Ingress Controller                              │
│                              Host: posts.com                                     │
│                                                                                  │
│   /posts/create ──► Posts Service       /posts ──► Query Service                 │
│   /posts/*/comments ──► Comments Service   /* ──► Client                         │
└──────────┬───────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                          Kubernetes Cluster                                       │
│                                                                                  │
│  ┌─────────────────┐                                                             │
│  │  Client (React)  │ :3000                                                      │
│  │  client-srv      │                                                            │
│  └────────┬─────────┘                                                            │
│           │  HTTP Requests                                                       │
│           ▼                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────┐        │
│  │                        API Layer (Write)                             │        │
│  │                                                                      │        │
│  │  ┌──────────────────┐          ┌─────────────────────┐              │        │
│  │  │  Posts Service    │          │  Comments Service    │              │        │
│  │  │  :4000            │          │  :4001               │              │        │
│  │  │                   │          │                      │              │        │
│  │  │ POST /posts/create│          │ POST /posts/:id/     │              │        │
│  │  │ GET  /posts       │          │      comments        │              │        │
│  │  └────────┬──────────┘          └──────────┬───────────┘              │        │
│  │           │                                │                          │        │
│  └───────────┼────────────────────────────────┼──────────────────────────┘        │
│              │  PostCreated                    │  CommentCreated                   │
│              │  Event                          │  CommentUpdated                   │
│              ▼                                 ▼                                   │
│  ┌──────────────────────────────────────────────────────────────────────┐        │
│  │                     Event Bus Service :4005                          │        │
│  │                     event-bus-srv                                     │        │
│  │                                                                      │        │
│  │   Receives events ──► Stores in memory ──► Broadcasts to ALL services│        │
│  │                                                                      │        │
│  │   Event Types:                                                       │        │
│  │   • PostCreated      • CommentCreated                                │        │
│  │   • CommentModerated • CommentUpdated                                │        │
│  └────────┬─────────────────────┬───────────────────────┬───────────────┘        │
│           │                     │                       │                         │
│           ▼                     ▼                       ▼                         │
│  ┌─────────────────┐  ┌─────────────────┐    ┌──────────────────┐               │
│  │ Moderation Svc  │  │  Query Service  │    │  Other Services  │               │
│  │ :4003           │  │  :4002          │    │  (Posts,Comments) │               │
│  │                 │  │                 │    │                   │               │
│  │ Filters comments│  │ Read Model:    │    │  Receive events   │               │
│  │ containing      │  │ Denormalized   │    │  to sync state    │               │
│  │ "orange"        │  │ Posts+Comments │    │                   │               │
│  │                 │  │                 │    │                   │               │
│  │ ──► Publishes   │  │ Replays events │    │                   │               │
│  │ CommentModerated│  │ on startup     │    │                   │               │
│  └─────────────────┘  └─────────────────┘    └──────────────────┘               │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Event Flow

```
 Create Post Flow                          Create Comment Flow
 ─────────────────                         ──────────────────────

 Client                                    Client
   │                                         │
   │ POST /posts/create                      │ POST /posts/:id/comments
   ▼                                         ▼
 Posts Service                             Comments Service
   │                                         │
   │ PostCreated                             │ CommentCreated
   ▼                                         ▼
 Event Bus ──broadcast──► Query Service    Event Bus ──broadcast──► Moderation
                                             │                        │
                                             │                        │ CommentModerated
                                             │                        ▼
                                             │◄──────────────────── Event Bus
                                             │
                                             │ (Updates status)
                                             │ CommentUpdated
                                             ▼
                                           Event Bus ──broadcast──► Query Service
```

## Tech Stack

| Layer            | Technology                   |
| ---------------- | ---------------------------- |
| Frontend         | React 18, Bootstrap 4, Axios |
| Backend          | Node.js, Express             |
| Communication    | HTTP/REST, Custom Event Bus  |
| Orchestration    | Kubernetes, Skaffold         |
| Containerization | Docker (Alpine Node images)  |
| Ingress          | NGINX Ingress Controller     |

## Services

| Service        | Port | Description                                        |
| -------------- | ---- | -------------------------------------------------- |
| **Client**     | 3000 | React SPA — create posts, view posts & comments    |
| **Posts**      | 4000 | Creates and stores blog posts                      |
| **Comments**   | 4001 | Creates comments, handles status updates           |
| **Query**      | 4002 | Read model — denormalized view of posts + comments |
| **Moderation** | 4003 | Validates comment content (rejects word "orange")  |
| **Event Bus**  | 4005 | Central event dispatcher and event store           |

## Project Structure

```
blog-boilerplate/
├── client/                  # React frontend
│   ├── src/
│   │   ├── App.js           # Main component
│   │   ├── PostCreate.js    # Post creation form
│   │   ├── PostList.js      # Posts display
│   │   ├── CommentCreate.js # Comment form
│   │   └── CommentList.js   # Comments display
│   ├── Dockerfile
│   └── package.json
├── posts/                   # Posts microservice
│   ├── index.js
│   ├── Dockerfile
│   └── package.json
├── comments/                # Comments microservice
│   ├── index.js
│   ├── Dockerfile
│   └── package.json
├── moderation/              # Moderation microservice
│   ├── index.js
│   ├── Dockerfile
│   └── package.json
├── query/                   # Query microservice
│   ├── index.js
│   ├── Dockerfile
│   └── package.json
├── event-bus/               # Event bus service
│   ├── index.js
│   ├── Dockerfile
│   └── package.json
├── infra/
│   └── k8s/                 # Kubernetes manifests
│       ├── client-depl.yaml
│       ├── posts-depl.yaml
│       ├── comments-depl.yaml
│       ├── query-depl.yaml
│       ├── moderation-depl.yaml
│       ├── event-bus-depl.yaml
│       └── ingress-srv.yaml
└── skaffold.yaml            # Local dev orchestration
```

## Getting Started

### Prerequisites

- [Docker](https://www.docker.com/)
- [Kubernetes](https://kubernetes.io/) (Docker Desktop K8s or Minikube)
- [Skaffold](https://skaffold.dev/)
- NGINX Ingress Controller

### Setup

1. **Enable Kubernetes** in Docker Desktop or start Minikube.

2. **Install NGINX Ingress Controller:**

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
   ```

3. **Add host entry** (for local development):

   ```bash
   # Add to /etc/hosts
   127.0.0.1 posts.com
   ```

4. **Start the application:**

   ```bash
   skaffold dev
   ```

5. **Open the app** at [http://posts.com](http://posts.com).

## Design Patterns

- **CQRS** — Separate write services (Posts, Comments) from the read service (Query)
- **Event Sourcing** — All state changes propagated as events through the Event Bus
- **Event Replay** — Query service replays full event history on startup to rebuild state
- **Eventual Consistency** — Read model is eventually consistent with write models

## API Endpoints

```
POST /posts/create          Create a new post        { title }
GET  /posts                 Get all posts + comments  (via Query service)
POST /posts/:id/comments    Add a comment             { content }
```

## Notes

- All services use **in-memory storage** — data is lost on service restart
- This is a **learning/demo project** illustrating microservices patterns
- For production use, add a persistence layer (MongoDB, PostgreSQL, etc.), retry logic, and proper error handling

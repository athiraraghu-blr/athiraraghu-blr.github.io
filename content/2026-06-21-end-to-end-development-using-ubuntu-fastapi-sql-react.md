Title: End-to-End Development Workflow Using Ubuntu, FastAPI, SQL, and React
Date: 2026-06-21
Category: Article
Tags: GenAI, LLM, machine-learning, deep-learning, announcement
Slug: ubuntu-fastapi-sql-react

Ubuntu, FastAPI, SQL, and React form a powerful full-stack workflow for building modern web applications from backend to frontend.

# Why This Stack

Each piece of this stack earns its place for specific reasons:

**Ubuntu** is the default operating system for most cloud servers and is widely supported by package managers, container tools, and CI/CD systems, making it a dependable foundation for both local development and production deployment.

**FastAPI** is a modern Python web framework built on async I/O and Python type hints. It automatically generates interactive API documentation (via OpenAPI/Swagger) and validates request/response data using Pydantic models, which speeds up backend development considerably.

**SQL databases** (PostgreSQL, MySQL, or SQLite for prototyping) provide reliable, structured, transactional storage — essential for applications with relational data like users, orders, or inventory.

**React** remains one of the most popular libraries for building interactive, component-based user interfaces, with a vast ecosystem of tooling and community support.

Together, these four layers form a clean separation of concerns: OS and infrastructure, data layer, API layer, and presentation layer.

# **Step 1: Setting Up the Ubuntu Environment**

Start by preparing a clean, reproducible development environment.

bash

sudo apt update && sudo apt upgrade -y

sudo apt install -y python3 python3-venv python3-pip \

    postgresql postgresql-contrib \

    nodejs npm \

    git curl build-essential

**A few practical notes:**

Use python3-venv to isolate Python dependencies per project rather than installing packages globally.

Install **Node.js via nvm** (Node Version Manager) instead of the default Ubuntu repository if you need to manage multiple Node versions across projects.

Set up **Git** early and configure SSH keys for your remote repository (GitHub, GitLab, etc.), since version control should track changes from the very first commit.

bash

curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

nvm install --lts

This base setup gives you a consistent environment that mirrors what you’d run in production, reducing “it works on my machine” issues later.

# **Step 2: Designing the SQL Database**

Before writing any backend code, design your schema. A relational database enforces structure and relationships that prevent data inconsistency.

Start PostgreSQL and create a database:

bash

sudo -u postgres createuser --interactive

sudo -u postgres createdb myapp_db

Define your schema with clear tables, primary keys, and foreign key relationships. For example, a simple task-management app might have:

sql

CREATE TABLE users (

    id SERIAL PRIMARY KEY,

    email VARCHAR(255) UNIQUE NOT NULL,

    hashed_password VARCHAR(255) NOT NULL,

    created_at TIMESTAMP DEFAULT NOW()

);

CREATE TABLE tasks (

    id SERIAL PRIMARY KEY,

    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,

    title VARCHAR(255) NOT NULL,

    is_complete BOOLEAN DEFAULT FALSE,

    created_at TIMESTAMP DEFAULT NOW()

);

Key practices at this stage:

Normalize data to avoid duplication, but don’t over-normalize to the point of hurting query performance.

Add indexes on columns you’ll frequently filter or join on.

Use migrations (via **Alembic**, which pairs naturally with FastAPI and SQLAlchemy) instead of hand-editing the schema, so changes are tracked and reversible.

# **Step 3: Building the Backend with FastAPI**

With the database ready, connect it to a FastAPI application.

**Project setup:**

bash

python3 -m venv venv

source venv/bin/activate

pip install fastapi uvicorn sqlalchemy psycopg2-binary alembic pydantic-settings

Define models with SQLAlchemy, mirroring your SQL schema, and Pydantic schemas to validate incoming and outgoing data:

python

#models.py

from sqlalchemy import Column, Integer, String, Boolean, ForeignKey

from database import Base

class Task(Base):

    __tablename__ = "tasks"

    id = Column(Integer, primary_key=True, index=True)

    user_id = Column(Integer, ForeignKey("users.id"))

    title = Column(String, nullable=False)

    is_complete = Column(Boolean, default=False)

python

#schemas.py

from pydantic import BaseModel

class TaskCreate(BaseModel):

    title: str

class TaskOut(TaskCreate):

    id: int

    is_complete: bool

    class Config:

        from_attributes = True

**Build routes** that expose clean, RESTful endpoints:

python

#main.py

from fastapi import FastAPI, Depends

from sqlalchemy.orm import Session

import models, schemas

from database import SessionLocal, engine

models.Base.metadata.create_all(bind=engine)

app = FastAPI()

def get_db():

    db = SessionLocal()

    try:

        yield db

    finally:

        db.close()

@app.post("/tasks/", response_model=schemas.TaskOut)

def create_task(task: schemas.TaskCreate, db: Session = Depends(get_db)):

    db_task = models.Task(title=task.title)

    db.add(db_task)

    db.commit()

    db.refresh(db_task)

    return db_task

@app.get("/tasks/", response_model=list[schemas.TaskOut])

def list_tasks(db: Session = Depends(get_db)):

    return db.query(models.Task).all()

Run the server locally:

bash

uvicorn main:app --reload

FastAPI automatically generates interactive documentation at /docs, which is invaluable for testing endpoints before the frontend even exists. This is also the point to add **CORS middleware** so your React app (running on a different port) can talk to the API:

python

from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(

    CORSMiddleware,

    allow_origins=["http://localhost:5173"],

    allow_methods=["*"],

    allow_headers=["*"],

)

# **Step 4: Building the Frontend with React**

With a working API, scaffold the frontend:

bash

npm create vite@latest frontend -- --template react

cd frontend

npm install axios

npm run dev

Structure components around the API resources you’ve built. A simple task list might look like this:

jsx

// TaskList.jsx

import { useEffect, useState } from "react";

import axios from "axios";

const API_URL = "http://localhost:8000";

export default function TaskList() {

  const [tasks, setTasks] = useState([]);

  const [title, setTitle] = useState("");

  useEffect(() => {

    axios.get(`${API_URL}/tasks/`).then((res) => setTasks(res.data));

  }, []);

  const addTask = async () => {

    const res = await axios.post(`${API_URL}/tasks/`, { title });

    setTasks([...tasks, res.data]);

    setTitle("");

  };

  return (

    <div>

      <input value={title} onChange={(e) => setTitle(e.target.value)} />

      <button onClick={addTask}>Add Task</button>

      <ul>

        {tasks.map((t) => (

          <li key={t.id}>{t.title}</li>

        ))}

      </ul>

    </div>

  );

}

Good practices at this layer:

Centralize API calls in a dedicated api.js or service layer instead of scattering axios calls across components.

Use environment variables (.env files with Vite’s VITE_ prefix) to switch the API base URL between development and production.

Handle loading and error states explicitly rather than assuming requests always succeed.

# **Step 5: Connecting the Pieces in a Local Dev Loop**

A productive workflow runs all three layers simultaneously:

PostgreSQL running as a background service (sudo systemctl start postgresql).

FastAPI running with --reload for instant backend updates.

React running with Vite’s dev server, which hot-reloads on save.

Many teams use a Makefile or a simple shell script to start everything at once, or move to **Docker Compose** once the project stabilizes, defining separate containers for the database, API, and frontend so the entire stack can be brought up with one command:

yaml

version: "3.9"

services:

  db:

    image: postgres:16

    environment:

      POSTGRES_DB: myapp_db

      POSTGRES_PASSWORD: postgres

    ports:

      - "5432:5432"

  backend:

    build: ./backend

    ports:

      - "8000:8000"

    depends_on:

      - db

  frontend:

    build: ./frontend

    ports:

      - "5173:5173"

This containerized setup also closes the gap between development and production, since the same Ubuntu-based images can be deployed to a cloud VM or Kubernetes cluster later.

# **Step 6: Testing**

A solid workflow includes tests at each layer:

1. **Backend**: Use pytest with FastAPI’s TestClient to test endpoints against a test database, verifying status codes, response shapes, and edge cases like invalid input.

2. **Database**: Test migrations apply cleanly in both directions (upgrade and downgrade) with Alembic.

3. **Frontend**: Use tools like **Vitest** and **React Testing Library** to test components in isolation, and **Cypress** or **Playwright** for end-to-end tests that simulate real user flows across the full stack.

# **Step 7: Deployment**

Once the application is stable:

1. **Provision an Ubuntu server** (a cloud VM from a provider like DigitalOcean, AWS EC2, or Linode).

2. **Set up PostgreSQL** on the server, or use a managed database service for easier backups and scaling.

3. **Run FastAPI behind a process manager** like systemd or supervisor, fronted by Nginx as a reverse proxy, with **Gunicorn + Uvicorn workers** handling concurrency:

bash

gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app

4. **Build the React app for production** (npm run build) and serve the static files via Nginx or a CDN.

5. **Automate deployment** with a CI/CD pipeline (GitHub Actions is a common choice) that runs tests, builds artifacts, and deploys on every merge to the main branch.

# **Step 8: Monitoring and Iteration**

After launch, the workflow doesn’t stop:

1. Use logging (structured JSON logs from FastAPI) and a tool like Sentry to catch backend and frontend errors in production.

2. Monitor database performance with EXPLAIN ANALYZE on slow queries and add indexes as usage patterns emerge.

3. Treat the development loop as continuous: new features move through the same path — schema change, API endpoint, frontend component, tests, deploy.

Conclusion

This Ubuntu–FastAPI–SQL–React workflow works well because each layer has a clear responsibility and a clean interface to the next: Ubuntu provides a consistent runtime, SQL enforces data integrity, FastAPI exposes that data through a fast, self-documenting API, and React turns it into an interactive experience. The real productivity gain comes not from any single tool but from the discipline of moving through the same loop — schema, API, UI, test, deploy — every time the application grows.
Title: Developing and Deploying a React & FastAPI Application on Ubuntu
Date: 2026-06-22
Category: Article
Tags: GenAI, LLM, machine-learning, deep-learning, announcement
Slug: developing-deploying-fastapi-react-ubuntu

Building modern web applications often means pairing a dynamic frontend with a robust backend API. React and FastAPI are a popular combination — React for its component-driven UI, and FastAPI for its speed, simplicity, and automatic API documentation. This article walks you through setting up, developing, and deploying both on an Ubuntu server.

# **Prerequisites**

Before diving in, make sure your Ubuntu system (20.04 or 22.04 LTS recommended) has the following installed:

1. **Node.js** (v18+) and **npm** — for the React frontend
2. **Python** (3.10+) and **pip** — for the FastAPI backend
3. **Nginx** — to serve as a reverse proxy
4. **Git** — for version control

# Install the basics in one go:

bash

    sudo apt update && sudo apt upgrade -y
    sudo apt install -y git nginx python3-pip python3-venv curl

    #Install Node.js via NodeSource
        curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
        sudo apt install -y nodejs

# **Part 1: Building the FastAPI Backend**

**1.1 Project Setup**

Create a project directory and set up a Python virtual environment:

bash

    mkdir ~/myapp && cd ~/myapp
    mkdir backend && cd backend
    python3 -m venv venv
    source venv/bin/activate
    pip install fastapi uvicorn

**1.2 Writing the API**

Create a file called main.py:

python

    from fastapi import FastAPI
    from fastapi.middleware.cors import CORSMiddleware
    app = FastAPI()
    app.add_middleware(
        CORSMiddleware,
        allow_origins=["*"],  # Restrict in production
        allow_methods=["*"],
        allow_headers=["*"],
    )
    @app.get("/")
    def root():
        return {"message": "Hello from FastAPI!"}
    @app.get("/items/{item_id}")
    def get_item(item_id: int):
        return {"item_id": item_id, "name": f"Item {item_id}"}

**1.3 Running the Backend Locally**

bash

    uvicorn main:app --reload --host 0.0.0.0 --port 8000

Visit http://localhost:8000/docs to see FastAPI's auto-generated Swagger documentation.

# **Part 2: Building the React Frontend**

**2.1 Create a React App**

From the ~/myapp directory:

bash

    cd ~/myapp
    npx create-react-app frontend
    cd frontend

**2.2 Fetching Data from the API**

Edit src/App.js to call your FastAPI backend:

javascript

    import { useEffect, useState } from "react";
    function App() {
        const [message, setMessage] = useState("Loading...");
            useEffect(() => {
                fetch("http://localhost:8000/")
                .then((res) => res.json())
                .then((data) => setMessage(data.message));
            }, []);
        return (
            <div style={{ padding: "2rem", fontFamily: "sans-serif" }}>
                <h1>React + FastAPI</h1>
                <p>{message}</p>
            </div>
        );
    }
    export default App;

**2.3 Running the Frontend Locally**

bash

    npm start

The React app will be available at http://localhost:3000.

# **Part 3: Deploying on Ubuntu**

**3.1 Build the React App**

When you're ready to deploy, produce a production build:

bash

    cd ~/myapp/frontend
    npm run build

This creates an optimized build/ folder ready to be served as static files.

**3.2 Run FastAPI with a Process Manager**

Install and configure **Gunicorn** with **Uvicorn workers** so the backend runs reliably:

bash
    
    cd ~/myapp/backend
    source venv/bin/activate
    pip install gunicorn

Create a **systemd service** to keep it running:

bash 

    sudo nano /etc/systemd/system/fastapi.service

ini

    [Unit]
    Description=FastAPI Application
    After=network.target
    [Service]
    User=ubuntu
    WorkingDirectory=/home/ubuntu/myapp/backend
    ExecStart=/home/ubuntu/myapp/backend/venv/bin/gunicorn -k uvicorn.workers.UvicornWorker main:app --bind 0.0.0.0:8000
    Restart=always
    [Install]
    WantedBy=multi-user.target

Enable and start the service:

bash

    sudo systemctl daemon-reload
    sudo systemctl enable fastapi
    sudo systemctl start fastapi

**3.3 Configure Nginx as a Reverse Proxy**

Create an Nginx configuration file:

bash

    sudo nano /etc/nginx/sites-available/myapp

nginx

    server {
        listen 80;
        server_name your-domain.com;  # or your server IP

        # Serve the React frontend
        location / {
            root /home/ubuntu/myapp/frontend/build;
            index index.html;
            try_files $uri /index.html;
        }

        # Proxy API requests to FastAPI
        location /api/ {
            proxy_pass http://127.0.0.1:8000/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }

Enable the site and restart Nginx:

bash

    sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
    sudo nginx -t
    sudo systemctl restart nginx

**3.4 (Optional) Enable HTTPS with Certbot**

bash

    sudo apt install -y certbot python3-certbot-nginx
    sudo certbot --nginx -d your-domain.com

Certbot will automatically configure SSL and set up certificate renewal.


# **Part 4: Updating Your Application**

When you push code changes, here's a simple update workflow:

bash

    #Pull latest code
    cd ~/myapp && git pull

    #Rebuild the frontend
    cd frontend && npm install && npm run build

    #Restart the backend service
    sudo systemctl restart fastapi

Summary

By following this guide, you've put together a solid, production-ready stack on Ubuntu. The React frontend is compiled into static files and served efficiently by Nginx on port 80, keeping page load times fast without any server-side rendering overhead. Behind the scenes, FastAPI runs on port 8000, powered by Uvicorn workers and managed by Gunicorn, with systemd ensuring the process restarts automatically if anything goes wrong.

Nginx acts as the glue between the two — routing ordinary browser requests to the React build and forwarding any calls to **/api/** directly to FastAPI. This means both your frontend and backend live under the same domain, avoiding cross-origin headaches in production. If you've added Certbot to the mix, all traffic is encrypted over HTTPS with certificates that renew themselves automatically.

The result is an architecture that's clean, maintainable, and easy to update — just pull your latest code, rebuild the frontend, and restart the backend service. From here, you can extend it further with Docker, CI/CD pipelines, or a managed database, but the foundation you've built is already well-suited for real-world use.
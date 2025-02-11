---
date: '2025-02-11T10:57:14+03:30'
draft: true
title: 'Nginx Docker Mulitiapp Project'
---
## Introduction

In this blog, I’m going to build a simple project using **Nginx** and **Docker**.  

Nginx is a widely used web server. It can function as a basic web server and also take on roles such as a **load balancer**, **reverse proxy**, and **content cache**.  

In this project, we’ll use **Nginx as a reverse proxy** for three simple web applications. A reverse proxy is essentially a web server that sits in front of other web servers and responds to client requests on their behalf. This setup has several advantages, including improved **performance, security, and reliability**.  

We have three different web applications, each serving a distinct purpose. They will be accessible under the same domain but with different URLs. If our domain is **example.com**, then:  

- **App 1** → **example.com/app1** (a static HTML file)  
- **App 2** → **example.com/app2** (a simple Node.js web app)  
- **App 3** → **example.com/app3** (a simple Flask web app)  

We will use Docker to **containerize all these applications**.  

By building this project, you can practice your **DevOps skills** and gain hands-on experience with **[Docker Compose](https://docs.docker.com/compose/), Dockerfiles**, and **networking between services in Docker**. You'll also learn how to manage and host multiple applications on a single server.  

This project isn’t just about building something—it’s about learning **real-world infrastructure management**, a highly valuable skill in today’s tech landscape. So, let's get started! 🚀  

---

## Step-by-Step Walkthrough

### Project Structure  

Your project folder will look like this:  

```project-root/
├── docker-compose.yml
├── README.md
├── nginx/
│   └── default.conf
├── app1/
│   ├── Dockerfile
│   └── index.html
├── app2/
│   ├── Dockerfile
│   ├── package.json
│   └── server.js
└── app3/
    ├── Dockerfile
    ├── requirements.txt
    └── app.py
```

### Step 1: Set Up the Directory Structure  

1. **Create the root folder:**  

```bash
mkdir nginx-docker-multiapp
cd nginx-docker-multiapp
```

2. **Create subdirectorie:**

```bash
mkdir nginx app1 app2 app3
```

---

### Step 2: Create the Docker Compose File
In this step, we’ll create a file named **docker-compose.yml** (or **compose.yaml** for newer versions) in the project root.

The first service in the compose file will be **Nginx**. We use the official Nginx image and mount our custom configuration file. This service depends on the three apps.

We also expose port **80** and define a shared network (`webnet`) for communication between services.

```yaml
services:
  nginx:
    image: nginx:latest
    container_name: nginx-proxy
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - app1
      - app2
      - app3
    networks:
      - webnet

  app1:
    build:
      context: ./app1
    container_name: app1
    networks:
      - webnet

  app2:
    build:
      context: ./app2
    container_name: app2
    networks:
      - webnet

  app3:
    build:
      context: ./app3
    container_name: app3
    networks:
      - webnet

networks:
  webnet:
    driver: bridge
```

At the end, we define the **webnet** network with the default bridge driver.

---

### Step 3: Configure Nginx reverse proxy
Create a file named **default.conf** inside the **nginx/** folder with the following content:

```nginx
server {
  listen 80;

  # Route for App1: static site
  location /app1/ {
    # Remove the /app1 prefix when forwarding to the app container
    proxy_pass http://app1:80/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    }

  # Route for App2: Node.js app
  location /app2/ {
    proxy_pass http://app2:3000/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    }

  # Route for App3: Flask app
  location /app3/ {
    proxy_pass http://app3:5000/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    }
}
```
This configuration tells Nginx to forward requests to the correct application based on the URL path.

**`proxy_pass`**: This directive forwards client requests to the appropriate backend service. 

**`proxy_set_header`**: These directives modify HTTP headers before forwarding the request. They ensure that the backend server receives important information, such as the original hostname and client IP.

* **`Host $host;`** – Preserves the original domain name instead of replacing it with the internal container hostname.
* **`X-Real-IP $remote_addr;`** – Passes the client's actual IP address, which is useful for logging and security.

For further reading, go to the documentation, [here](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_set_header).

---

### Step 4: Build each application
**App 1 - Static website**

Create `index.html` inside the `app1/` directory:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>App 1 - Static Site</title>
</head>
<body>
  <h1>Hello from App 1 (Static Site)!</h1>
  <p>This page is served as a static site via an Nginx container.</p>
</body>
</html>
```
Create a `Dockerfile` inside `app1/`:
```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
```

**App 2 - Node.js Application**

Create `server.js` inside `app2/`:

```javascript
const http = require('http');
const port = 3000;

const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello from App 2 (Node.js)!\n');
});

server.listen(port, () => {
  console.log(`App 2 is running on port ${port}`);
});
```

**App 3 - Python Flask Application**

Create requirements.txt inside app3/:
```text
Flask==2.3.2
```

Create `app.py` inside `app3/`:
```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello from App 3 (Flask)!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

Create a `Dockerfile` inside `app3/`:
```dockerfile
FROM python:3.8-slim-buster
WORKDIR /app
COPY requirements.txt requirements.txt
RUN pip3 install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python3", "app.py"]
```
---

### Step 5: Build and Run the Project
We can build and start all services using **Docker Compose** with a single command:
```bash
docker compose up --build -d
```
This will:

1. Build the necessary images
2. Create the network
3. Start all services

### Step 6: Test the Setup
Once everything is up and running, open your browser and visit:

* **http://localhost/app1/** → You should see the static site.
* **http://localhost/app2/** → You should see the Node.js message.
* **http://localhost/app3/** → You should see the Flask message.

To check logs:
```bash
docker compose logs -f
```

To stop services:
```bash
docker compose down
```

---

## Conclusion

By completing this project, you've gained hands-on experience in setting up a **multi-app environment** using **Docker** and **Nginx as a reverse proxy**. You’ve learned how to:  

✅ Containerize different types of applications (Static HTML, Node.js, and Flask).  
✅ Use **Docker Compose** to manage multiple services.  
✅ Configure **Nginx** to route traffic efficiently between apps.  
✅ Establish **networking between containers** to enable seamless communication.  

This setup mirrors real-world infrastructure scenarios, where multiple applications run on a single server while remaining independent and scalable. You can extend this project by adding **SSL certificates** (using Let’s Encrypt), implementing **load balancing**, or integrating a **database** for dynamic content.  

If you found this useful, consider pushing your project to **GitHub** and sharing it with others! 🚀  

You can also clone this project's repository to your machine:
```bash
git clone https://github.com/ahmadpiran/nginx-docker-multiapp
```

Now, it’s time to take our DevOps journey even further. Keep building, keep coding! 🎉




---
date: '2025-02-24T18:30:00+03:30'
draft: false
title: 'Setting Up Your Self-Hosted GitLab and Registering a GitLab Runner using Docker: A Step-by-Step Guide'
tags: ["gitlab", "docker", "project", "CI/CD"]
---
## Introduction
In this blog post, I’ll walk you through 👣 setting up a self-hosted GitLab instance and registering a GitLab runner. If you are already familiar with GitLab, you’re in the right place. If not, no worries; I recommend checking out [this introductory video](https://www.youtube.com/watch?v=0pOvg8QkKiw) to get up to speed.

While you can easily sign up at [gitlab.com](https://gitlab.com/)🌐 and use their hosted services, self-hosting GitLab offers you a whole new level of **control**—plus It’s **free**! Whether you’re creating a personal lab for your learning journey 📚 or your team needs a private, full-featured code repository and DevOps solution, this guide is for you.

Today, we’re going to set up GitLab on a local machine using **Docker**🐳. Once GitLab is up and running, we’ll register a GitLab runner to power your **CI/CD** pipelines. Think of a runner as your friendly little helper that executes your pipeline jobs. So, grab a cup of coffee☕, sit back, and let’s dive into this exciting setup together!

## Prerequisites
For this demonstration, we’ll deploy the GitLab instance and the runner on a personal or local machine. Please ensure you have at least:
- **16 GB of memory**
- **4 CPU Cores**
- **10 GB of free storage**

You’ll also need a working Docker setup. **Note:** Docker for windows isn’t officially supported for this setup due to potential compatibility challenges with volume permissions and other issues.

If you’re planning to set this up for production, please refer to the [GitLab installation requirements](https://docs.gitlab.com/install/requirements/). 

## Setting up self-hosted GitLab
GitLab’s core functionality is to serve as a code repository. To interact with the repository via SSH, we need to configure SSH. Since the GitLab container will run on our local machine, we must configure SSH to use a different port instead of its default port (22). In this guide, I’ve chosen port **2424**.

We also need to persist GitLab data on our local machine so that stopping and restarting the container won’t cause data loss. We’ll create directories for configuration files, logs, and data.

In your project’s root directory, create the following folder:

```bash
mkdir -p srv/gitlab
```
Next, create a Docker Compose file in your root directory:

```bash
touch compose.yaml
```

Then add the following configuration to compose.yaml:
```yaml
services:
  gitlab:
    image: gitlab/gitlab-ee
    container_name: gitlab
    hostname: "gitlab.example.com"
    ports:
      - "8090:80" # HTTP access
      - "2424:22" # SSH access
    volumes:
      - './srv/gitlab/config:/etc/gitlab'
      - './srv/gitlab/logs:/var/log/gitlab'
      - './srv/gitlab/data:/var/opt/gitlab'
    networks:
      - mylab_net

  gitlab-runner:
    image: gitlab/gitlab-runner
    container_name: gitlab-runner
    depends_on:
      - gitlab
    volumes:
      - './srv/gitlab-runner/config:/etc/gitlab-runner'
      - /var/run/docker.sock:/var/run/docker.sock # Host Docker socket
    networks:
      - mylab_net

networks:
  mylab_net:
    driver: bridge
```

**A Few Notes:**
- **SSH Port:** We map host port 2424 to container port 22 to avoid conflicts.
- **Persistent Storage:** The volumes for the GitLab service ensure that configuration, logs, and data are saved on your host.
- **Networking:** We create a custom bridge network (mylab_net) so that GitLab and GitLab Runner can communicate.
- **hostname:** We are using `example.com` domain here. To ensure everything works smoothly, you’ll need to add a record to your local machine’s `/etc/hosts` file. Simply include this line:
```
[YOUR-MACHINE-IP] gitlab.example.com
```
Replace `[YOUR-MACHINE-IP]` in the example below with your local machine’s actual IP address (check it using `ifconfig` or `ipconfig`!).

## Installing GitLab Runner
GitLab runner isn’t included in the GitLab image, so we install it separately using Docker. The official image is gitlab/gitlab-runner. In the compose file above, I’ve already added the GitLab Runner service.

To ensure that the runner’s configuration isn’t lost when the container restarts, create a directory for it in your project’s root:
```bash
mkdir srv/gitlab-runner
```

### How the Runner works
On the GitLab docs we have:
> GitLab Runner is an application that works with GitLab CI/CD to run jobs in a pipeline.

In our Lab, we set up the GitLab runner as a Docker container. This container must be able to run executors. Executors define the environment in which CI/CD jobs are executed. They determine how tasks are run. There are different types of [executors](https://docs.gitlab.com/runner/executors/).

In our case, we use Docker as executor. So, we need a way to run Docker containers as executors in our GitLab runner container. Instead of running Docker-in-Docker(which can add complexity), we mount the host’s [Docker socket](https://stackoverflow.com/questions/35110146/what-is-the-purpose-of-the-file-docker-sock) (/var/run/docker.sock) into the runner container. This allows the runner to communicate directly with the host’s Docker daemon.

## Starting the Setup
Now that your `compose.yaml` file is ready, start the services by running:
```bash
docker compose up -d
```
**Note:** This process might take a while since the GitLab image is quite large. Once everything is up, open your browser and navigate to http://gitlab.example.com

## Accessing Your GitLab Instance
- **Default user:** root
- **Initial password:** The password is generated automatically. You can retrieve it by running:
```bash
docker exec -it gitlab grep 'Password:'
```

that gives you the `/etc/gitlab/initial_root_password` file.

**Important:** After logging in, change the root password. The initial password file is automatically deleted after  24 hours.

## Registering GitLab Runner with Your Self-Hosted GitLab
Once your GitLab server is up and running, follow these steps to register your GitLab Runner:
- **Log In:** Open your GitLab instance in a browser and log in as the root user.
- **Navigate to Runners:** On the left-hand sidebar, click on **CI/CD** and then **Runners**.
- **Add a New Runner:** Click on **Add new runner**. Review the available options (they’re pretty self-explanatory) and click **Create runner**.
- **Select the Operating System:** On the next page, choose **Linux** as the operating system.
- **Copy the Registration Command:** In Step 1, you’ll see a registration command. Copy this command.
- **Register the Runner:** Open a terminal on your host machine. Run the following command to open a shell in the GitLab Runner container:
```bash
docker container exec -it gitlab-runner bash
```
Paste the registration command inside the container’s shell and follow the prompts to complete the registration.

## Updating the GitLab Runner Configuration
After registering the runner, it’s important to ensure it communicates with the GitLab instance using the correct URLs. To do this, we need to update the runner’s configuration file. Open the file located at `srv/gitlab-runner/config/config.toml` and add the following lines below the `[[runners]]`:
```
url = "http://[YOUR-MACHINE-IP]:8090"
clone_url = "http://[YOUR-MACHINE-IP]:8090"
``` 

**What These Settings Do**

- **url:** This line tells the GitLab Runner the base URL of your GitLab server. It’s the endpoint the runner will use to communicate with GitLab for tasks like fetching job details and sending back results.
- **clone_url:** This setting specifies the URL the runner should use when cloning repositories. This is particularly useful if your GitLab instance is accessible via a non-standard port or a custom domain.

Once ’ve updated the configuration file, restart your GitLab Runner container to apply the changes:
```bash
docker container restart gitlab-runner
```

Now, the runner should be fully configured and ready to seamlessly interact with the GitLab instance using the specified URLs.

## Conclusion
Woohoo! 🎉 We’ve just set up a self-hosted GitLab instance using Docker 🐳 and registered a GitLab Runner to tackle all our CI/CD jobs! Now, we’ve got your very own powerhouse DevOps environment right at our fingertips.

Whether you’re tinkering with personal projects, collaborating with a team, or just love having total control over your tools, this setup is your golden ticket. Self-hosting isn’t just about flexibility—it’s about making tech work for you.

Go ahead—automate, deploy, and innovate with confidence! 🚀💻

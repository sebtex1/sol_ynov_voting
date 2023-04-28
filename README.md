# Ynov Voting App

A simple distributed application running across multiple Docker containers.

## Getting started

This application uses Python, Node.js, .NET, with Redis for messaging and Postgres for storage.

The `vote` app will be running at [http://localhost:5005](http://localhost:5000), and the `results` will be at [http://localhost:5001](http://localhost:5001).
Feel free to use differents ports

## Architecture

![Architecture diagram](architecture.excalidraw.png)

- A front-end web app in [Python](/vote) which lets you vote between two options
- A [Redis](https://hub.docker.com/_/redis/) which collects new votes
- A [.NET](/worker/) worker which consumes votes and stores them in…
- A [Postgres](https://hub.docker.com/_/postgres/) database backed by a Docker volume
- A [Node.js](/result) web app which shows the results of the voting in real time

## Notes

The voting application only accepts one vote per client browser. It does not register additional votes if a vote has already been submitted from a client.

# You will be evaluated based on below tasks

Before starting working on the project make sure to fork it first.

## TODO: Dockerfile

Create a dockerfile for each project

1. result

```shell
FROM node:18-slim

# add curl for healthcheck
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
    curl \
    tini \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# have nodemon available for local dev use (file watching)
RUN npm install -g nodemon

COPY package*.json ./

RUN npm ci \
 && npm cache clean --force \
 && mv /app/node_modules /node_modules

COPY . .

ENV PORT 80
EXPOSE 80

ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["node", "server.js"]
```

2. vote

```shell
# Using official python runtime base image
FROM python:3.9-slim

# add curl for healthcheck
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Set the application directory
WORKDIR /app

# Install our requirements.txt
COPY requirements.txt /app/requirements.txt
RUN pip install -r requirements.txt

# Copy our code from the current folder to /app inside the container
COPY . .

# Make port 80 available for links and/or publish
EXPOSE 80

# Define our command to be run when launching the container
CMD ["gunicorn", "app:app", "-b", "0.0.0.0:80", "--log-file", "-", "--access-logfile", "-", "--workers", "4", "--keep-alive", "0"]

```

2. worker

```shell
# because of dotnet, we always build on amd64, and target platforms in cli
# dotnet doesn't support QEMU for building or running.
# (errors common in arm/v7 32bit) https://github.com/dotnet/dotnet-docker/issues/1537
# https://hub.docker.com/_/microsoft-dotnet
# hadolint ignore=DL3029
# to build for a different platform than your host, use --platform=<platform>
# for example, if you were on Intel (amd64) and wanted to build for ARM, you would use:
# docker buildx build --platform "linux/arm64/v8" .
FROM --platform=${BUILDPLATFORM} mcr.microsoft.com/dotnet/sdk:7.0 as build
ARG TARGETPLATFORM
ARG BUILDPLATFORM
RUN echo "I am running on $BUILDPLATFORM, building for $TARGETPLATFORM"

WORKDIR /source
COPY *.csproj .
RUN case ${TARGETPLATFORM} in \
         "linux/amd64")  ARCH=x64  ;; \
         "linux/arm64")  ARCH=arm64  ;; \
         "linux/arm64/v8")  ARCH=arm64  ;; \
         "linux/arm/v7") ARCH=arm  ;; \
    esac \
    && dotnet restore -r linux-${ARCH}

COPY . .
RUN  case ${TARGETPLATFORM} in \
         "linux/amd64")  ARCH=x64  ;; \
         "linux/arm64")  ARCH=arm64  ;; \
         "linux/arm64/v8")  ARCH=arm64  ;; \
         "linux/arm/v7") ARCH=arm  ;; \
    esac \
    && dotnet publish -c release -o /app -r linux-${ARCH} --self-contained false --no-restore

# app image
FROM mcr.microsoft.com/dotnet/runtime:7.0
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "Worker.dll"]
```

4. seed-data

```shell
FROM python:3.9-slim

# add apache bench (ab) tool
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
    apache2-utils \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /seed

COPY . .

# create POST data files with ab friendly formats
RUN python make-data.py

CMD /seed/generate-votes.sh
```

## TODO: Build images

- Create a `docker-compose-build.yaml` file that will build all images
- Run `docker-compose -f docker-compose-build.yaml build --parallel` to create images
- Publish the images to dockerhub
- Put a screenshot of your dockerhub images inside `screenshots` folder

## TODO: Create a containers

- A sample `docker-compose.yaml` file is available in the root folder. Use it to create all the containers in one go.
- Put a screenshot of running containers inside `screenshots` folder

## TODO: Virtual machines and project setup

1. Create 5 virtual machines

   - `gitlab-instance` will host a running instance of gitlab application
   - `gitlab-runner-shell` will host a shell based runner
   - `gitlab-runner-docker` will host a docker executor based runner
   - `dev-server` will deploy the ynov-voting application
   - `prod-server` will deploy the ynov-voting application

Note: The images tags deployed on `dev-server` and `prod-server` need to be differents.

2. Put a screenshot of the virtual machines inside `screenshots` folder
3. Create a project inside gitlab
4. Apart from the creator of gitlab repo, create 2 other users as part of your team and add them with Developer role to the repository
5. Fork the github voting project and push to the gitlab repository. Make sure to set the appropriate repository url
6. Take all necessary screenshots that demonstrate all the tasks and put them inside the `screenshots` folder

## TODO: CICD Pipeline

1. Create a file named `.gitlab-ci.yml` inside the root project folder that will contains the CICD pipeline
2. The pipeline should conntains 3 stages
   - `build-n-push-images` This stage will build images and push them to dockerhub
   - `deploy-dev` This stage will run a job that will deploy images to the `dev-server`
   - `deploy-prod` This stage will run a job that will deploy images to the `prod-server`
3. From the deployment servers, and using any container, run `docker inspect <container-name> | grep image`. This command will show the deployed image. The goal of this task is to make sure the two servers have differents images tag deployed.
4. The 2 other members need to create feature branch and suggest a small changes to the project. It can be a simple README.md update. The need to create a merge request so that the owner of the repo can merge to master branch
5. Put the necessary screenshots that demonstrate your work
6. Make sure to reset your local project url to a github repo and push all changes on the github repo as this is the one your are going to submit

## RUNNING APPLICATION

As final task, access the application using the deployment server public ip and take a screenshot of the voting application. Feel free to play with the application by doing some vote for cats or dogs :)
PLEASE DO NOT FORGET TO ENTER YOUR GITHUB URL AND YOUR TEAM MEMBERS NAMES when submitting your work

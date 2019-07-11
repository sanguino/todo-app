# Copies in our code and runs NPM Install
FROM node:latest AS base
WORKDIR /usr/src/app
ADD https://api.github.com/repos/sanguino/todo-front/git/refs/heads/master ../version.json
RUN git clone git://github.com/sanguino/todo-front.git .
RUN npm ci

# Lints Code
FROM base AS linting
WORKDIR /usr/src/app
COPY --from=base /usr/src/app/ .
RUN npm run lint

# Gets Sonarqube Scanner from Dockerhub and runs it
#FROM newtmitch/sonar-scanner:latest AS sonarqube
#COPY --from=base /usr/src/app/src /usr/src
#RUN sonar-scanner -Dsonar.projectBaseDir=/usr/src

# Runs Unit Tests
FROM node-chrome:latest AS unit-tests
WORKDIR /usr/src/app
COPY --from=base /usr/src/app/ .
RUN npm run test

# Runs Build
FROM base AS build
WORKDIR /usr/src/app
COPY --from=base /usr/src/app/ .
RUN npm run build

# Starts and Serves Web Page
FROM nginx:stable
COPY --from=build /usr/src/app/dist /usr/share/nginx/html

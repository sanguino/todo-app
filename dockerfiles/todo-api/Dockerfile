# Copies in our code and runs NPM Install
FROM node:latest AS base
WORKDIR /usr/src/app
RUN git clone git://github.com/sanguino/todo-api.git .
RUN npm ci

# Lints Code
FROM base AS linting
WORKDIR /usr/src/app
COPY --from=base /usr/src/app/ .
RUN npm run lint

# Gets Sonarqube Scanner from Dockerhub and runs it
FROM newtmitch/sonar-scanner:latest AS sonarqube
COPY --from=base /usr/src/app/src /usr/src
CMD sonar-scanner -Dsonar.projectBaseDir=/usr/src

# Runs Unit Tests
FROM base AS unit-tests
WORKDIR /usr/src/app
COPY --from=base /usr/src/app/ .
RUN npm run test

# Starts and Serves Web Page
FROM node:latest
COPY --from=base /usr/src/app/src /usr/src
RUN node start.js

# Copies in our code and runs NPM Install
FROM node:latest AS base
WORKDIR /usr/src/app
ADD https://api.github.com/repos/sanguino/todo-auth/git/refs/heads/master ../version.json
RUN git clone git://github.com/sanguino/todo-auth.git .
RUN npm ci
RUN npm run lint
RUN npm run test

# Starts and Serves Web Page
FROM node:latest
WORKDIR /usr/src/app
COPY --from=base /usr/src/app .
CMD node start.js

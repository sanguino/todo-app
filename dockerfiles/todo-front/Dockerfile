# Copies in our code and runs NPM Install
FROM node-chrome:latest AS build
WORKDIR /usr/src/app
ADD https://api.github.com/repos/sanguino/todo-front/git/refs/heads/master ../version.json
RUN git clone git://github.com/sanguino/todo-front.git .
RUN npm ci
RUN npm run lint
RUN npm run test
RUN npm run build

# Starts and Serves Web Page
FROM nginx:stable
COPY --from=build /usr/src/app/dist /usr/share/nginx/html

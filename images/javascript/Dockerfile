FROM node:20.9-bullseye

WORKDIR /app

SHELL ["/bin/bash", "-c"]

ONBUILD COPY package.json /app
ONBUILD RUN npm install
ONBUILD COPY . /app

ONBUILD CMD ["node", "."]
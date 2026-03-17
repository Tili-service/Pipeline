FROM node:20.9-bullseye

WORKDIR /app

SHELL ["/bin/bash", "-c"]

ONBUILD COPY package.json package-lock.json /app
ONBUILD RUN npm ci
ONBUILD COPY . .

ONBUILD EXPOSE 3000
ONBUILD CMD ["npm", "run", "dev"]

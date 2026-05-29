FROM node:20-alpine

WORKDIR /app

COPY package.json .
COPY index.js .

RUN npm install

USER node

EXPOSE 3000

CMD ["node", "index.js"]

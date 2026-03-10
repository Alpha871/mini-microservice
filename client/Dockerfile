From node:alpine

ENV CI=true
ENV WDS_SOCKET_PORT=0

WORKDIR /app


COPY package.json ./

RUN npm install

COPY . .

# EXPOSE 4000

CMD ["npm", "start"]
#---------Stage 1:Builder-------------

FROM node:24 AS builder

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

RUN npm run build

#-----------Stage 2:Runner--------------

FROM nginx:alpine

WORKDIR /app

COPY --from=builder /app/dist /usr/share/nginx/html

CMD ["nginx","-g","daemon off;"]

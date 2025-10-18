# package install for production 
FROM node:18-alpine AS base 
WORKDIR /app 
COPY package*.json ./ 
RUN npm i --only=production && npm cache clean --force 

# package install for development 
FROM node:18-alpine AS dev-deps
WORKDIR /app 
COPY package*.json ./ 
RUN npm i 

# build stage 
FROM node:18-alpine AS build 
WORKDIR /app 
COPY --from=dev-deps /app/node_modules ./node_modules
COPY . .

FROM node:18-alpine AS production
WORKDIR /app

RUN addgroup -S nodejs -g 1001 && \ 
    adduser -S backend -u 1001 

COPY --from=base /app/node_modules ./node_modules
COPY --from=build --chown=backend:nodejs /app .

USER backend 

EXPOSE 3001 

CMD ["node", "index.js"]
FROM node:22-alpine

WORKDIR /app

COPY . /app/

RUN npm install -g 12306-mcp

CMD ["npx", "-y", "12306-mcp", "--port", "3000"]


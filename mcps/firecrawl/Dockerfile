FROM node:22-alpine

WORKDIR /app

COPY . /app/

RUN npm install -g firecrawl-mcp

CMD ["npx", "-y", "firecrawl-mcp"]

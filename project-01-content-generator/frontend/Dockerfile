# Multi-stage build for production frontend
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install ALL dependencies (needed for build)
RUN npm install

# Copy source code
COPY . .

# Build the React app
RUN npm run build

# Production stage
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install ONLY production dependencies
RUN npm install --only=production

# Copy server.js
COPY server.js ./

# Copy built React app from builder stage
COPY --from=builder /app/build ./build

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup && \
    chown -R appuser:appgroup /app

USER appuser

# Cloud Run will set PORT=8080
ENV PORT=8080
ENV NODE_ENV=production

EXPOSE 8080

# Start the proxy server
CMD ["npm", "run", "serve"]

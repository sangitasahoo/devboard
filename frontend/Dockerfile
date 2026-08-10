# Stage 1 — build the app
FROM dhi.io/node:26-dev AS build

WORKDIR /app

COPY package*.json ./

RUN npm install --legacy-peer-deps

# Install vite explicitly if not already in package.json
RUN npm list vite > /dev/null 2>&1 || npm install --no-save vite@5.4.11

COPY . .

RUN npm run build

# Stage 2 — distroless runtime (minimal, secure)
FROM dhi.io/node:26

WORKDIR /app

# Create non-root user
RUN useradd -m -u 1000 appuser

# Copy built files from builder
COPY --from=build /app/dist ./dist

# Copy all node_modules from builder (includes vite)
COPY --from=build /app/node_modules ./node_modules

COPY vite.preview.config.js ./vite.config.js

# Fix permissions for non-root user
RUN chown -R appuser:appuser /app

USER appuser

EXPOSE 5173

# Health check for the app
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:5173', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})" || exit 1

# Use exec form for proper signal handling
ENTRYPOINT ["node", "--no-warnings"]
CMD ["./node_modules/.bin/vite", "preview", "--host", "0.0.0.0", "--port", "5173"]

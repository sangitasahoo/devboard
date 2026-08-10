# Stage 1 — build the app
FROM dhi.io/node:26-dev AS build

WORKDIR /app

COPY package*.json ./

RUN npm install --legacy-peer-deps

# Install vite explicitly if not already in package.json
RUN npm list vite > /dev/null 2>&1 || npm install --no-save vite@5.4.11

COPY . .

RUN npm run build

# Stage 2 — prune devDependencies (removes esbuild, vite, etc.)
FROM dhi.io/node:26-dev AS pruned

WORKDIR /app

COPY package*.json ./
COPY --from=build /app/node_modules ./node_modules

# Prune devDependencies to remove esbuild, vite, and other build tools
RUN npm prune --omit=dev --force

# Stage 3 — distroless runtime (minimal, secure)
FROM dhi.io/node:26

WORKDIR /app

# Copy built dist
COPY --from=build /app/dist ./dist

# Copy only production dependencies (esbuild, vite excluded)
COPY --from=pruned /app/node_modules ./node_modules

# Copy package.json for reference
COPY --from=build /app/package.json ./

EXPOSE 5173

# Use exec form for proper signal handling
ENTRYPOINT ["node", "--no-warnings"]
CMD ["./node_modules/.bin/vite", "preview", "--host", "0.0.0.0", "--port", "5173"]

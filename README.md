# 🚀 NewsoftDS API Server Buildpack

**Optimized Heroku buildpack for deploying API servers from Nx monorepos with aggressive size reduction.**

This is a specialized version of the main NewsoftDS buildpack, optimized specifically for API deployments where slug size is critical.

## ✨ What Makes This Different

### Compared to the Standard Buildpack:

| Feature | Standard Buildpack | API Buildpack |
|---------|-------------------|---------------|
| Target Use Case | General monorepo apps | API servers only |
| Slug Size Focus | Moderate | **Extreme** |
| node_modules Cleanup | Basic | **Aggressive** |
| Monorepo Isolation | Partial | **Complete** |
| Production Deps | Merged with monorepo | **Fresh install only** |

### Key Optimizations:

1. **🧹 Complete Monorepo Isolation**
   - Deletes 3.0G+ monorepo `node_modules` before production install
   - Fresh production-only dependency install (typically 50-150M)
   - **Saves ~2.5-3.0G** in slug size

2. **🔥 Aggressive node_modules Cleanup**
   - Removes test files and directories
   - Strips documentation (*.md, CHANGELOG, LICENSE)
   - Deletes TypeScript source (keeps .d.ts for types)
   - Removes examples and demo code
   - Strips source maps in production
   - Removes dev configs (.eslintrc, .prettierrc, etc.)
   - **Saves additional 30-50%** of node_modules size

3. **📦 Smart Build Process**
   - Uses project's `build:heroku` script
   - Copies only compiled output to deployment root
   - Minimal runtime dependencies (only what's externalized)
   - No Prisma generation (uses pre-committed client)

## 🎯 Perfect For:

- ✅ GraphQL API servers
- ✅ REST API servers
- ✅ Microservices
- ✅ Any Node/Bun API with Heroku's 500M slug limit

## 📋 Requirements

1. **Nx monorepo** with `generatePackageJson` enabled
2. **Project has `build:heroku` script** that outputs to `dist/`
3. **Prisma client is pre-generated** and committed (if using Prisma)
4. **Minimal external dependencies** defined in build script

## 🚀 Setup

### 1. Configure Your Project

Ensure your API server has these in `package.json`:

```json
{
  "scripts": {
    "build:heroku": "bun ./src/scripts/build-heroku.ts",
    "start": "bun ./dist/index.js"
  }
}
```

Your `build-heroku.ts` should:
- Build your TypeScript to `dist/index.js`
- Copy Prisma schema and generated client to `dist/`
- Generate minimal `package.json` with only essential external deps
- Copy any runtime assets (public/, etc.)

### 2. Set Heroku Environment Variables

```bash
# Required
heroku config:set BP_BUILD=apis/servers/api-client-server
heroku config:set BP_START=start

# Optional (recommended defaults)
heroku config:set BP_CLEAN=true
heroku config:set BP_NODE=false
```

### 3. Add Buildpack to Heroku

**Option 1: Via Heroku CLI**
```bash
heroku buildpacks:set https://github.com/your-org/NewsoftDS-Monorepo.git#main:utils/buildpack-api
```

**Option 2: Via app.json**
```json
{
  "buildpacks": [
    {
      "url": "https://github.com/your-org/NewsoftDS-Monorepo.git#main:utils/buildpack-api"
    }
  ]
}
```

### 4. Deploy

```bash
git push heroku main
```

## 📊 Expected Results

### Before (Standard Buildpack):
```
Compiled slug size: 691.4M
❌ Too large (max is 500M)
```

### After (API Buildpack):
```
Compiled slug size: 150-250M
✅ Under 500M limit
```

## 🔧 Environment Variables

### Required

| Variable | Description | Example |
|----------|-------------|---------|
| `BP_BUILD` | Project path or package name | `apis/servers/api-client-server` |
| `BP_START` | Start command (script name or command) | `start` or `bun ./index.js` |

### Optional

| Variable | Default | Description |
|----------|---------|-------------|
| `BP_CLEAN` | `true` | Enable aggressive cleanup |
| `BP_NODE` | `false` | Install Node.js (usually not needed with Bun) |
| `BP_NODE_VERSION` | `22.11.0` | Node.js version if BP_NODE=true |
| `BP_BUN_VERSION` | `latest` | Bun version to install |
| `BP_INSTALL` | `bun install` | Command to install monorepo deps |

## 📝 Build Flow

```
1. Install Bun (and optionally Node.js)
   ↓
2. Install monorepo dependencies (with --ignore-scripts)
   ↓
3. Build your project (runs build:heroku script)
   → Creates dist/ with:
     - index.js (bundled server code)
     - package.json (minimal production deps)
     - prisma/ (schema + pre-generated client)
     - public/ (static assets)
   ↓
4. Copy dist/* to deployment root
   ↓
5. 🔥 DELETE monorepo node_modules (3.0G)
   ↓
6. Fresh install of production deps only (~50-150M)
   ↓
7. Aggressive cleanup:
   - Remove monorepo source (apis/, web/, etc.)
   - Strip node_modules (tests, docs, examples, etc.)
   - Remove build artifacts (.nx, .cache, etc.)
   - Remove .git directory
   - Remove Bun install cache
   ↓
8. Create Procfile with start command
   ↓
✅ Deploy slug (~150-250M)
```

## 🎯 Key Differences from Standard Buildpack

### Standard Buildpack (utils/buildpack):
```bash
# Line 257-262: Install production deps
rm -f bun.lockb bun.lock package-lock.json yarn.lock
# ❌ Doesn't remove node_modules
bun install --production
# Result: Merges with 3.0G monorepo node_modules = 3.2G slug
```

### API Buildpack (utils/buildpack-api):
```bash
# Line 257-269: Install production deps
rm -rf node_modules  # ✅ Removes 3.0G monorepo node_modules
rm -f bun.lockb bun.lock package-lock.json yarn.lock
bun install --production
# Result: Fresh 50-150M production deps = ~200M slug
```

## 🔍 Troubleshooting

### Slug still too large

Check the build logs for:
```
ℹ️  Largest directories before cleanup:
3.0G    node_modules  # ❌ Should NOT be this large
```

If you see this, ensure:
1. You're using the **API buildpack**, not the standard one
2. `BP_CLEAN=true` is set
3. Your build script only externalizes essential packages

### Missing dependencies at runtime

If your app crashes with "Cannot find module":
1. Check your `build-heroku.ts` externalizes that package
2. Ensure it's in the generated `package.json` in `dist/`
3. Verify it's listed in root `package.json` catalog

### Prisma client not found

This buildpack expects Prisma client to be **pre-generated and committed**:
```bash
# Run locally
bun prisma:generate

# Commit the generated client
git add prisma/generated
git commit -m "Add pre-generated Prisma client"
git push heroku main
```

## 📈 Size Comparison

### What Gets Included:

| Item | Size | Notes |
|------|------|-------|
| Bun runtime | ~100M | In `.heroku/bin/` |
| Production node_modules | 50-150M | Only externalized packages |
| Compiled server code | 5-10M | Bundled with dependencies |
| Prisma schema + client | 50-100M | Pre-generated |
| Public assets | 1-5M | Static files |
| **Total Slug** | **~200-350M** | ✅ Well under 500M limit |

### What Gets Removed:

| Item | Size Saved | How |
|------|------------|-----|
| Monorepo node_modules | ~2.5-3.0G | Deleted before production install |
| Source directories | ~200-300M | Removed after build |
| Tests in node_modules | ~50-100M | Aggressive cleanup |
| Docs in node_modules | ~20-50M | Aggressive cleanup |
| Examples in node_modules | ~10-30M | Aggressive cleanup |
| .git directory | ~100-500M | Removed after build |
| Build artifacts | ~50-100M | Nx cache, tmp files, etc. |
| **Total Saved** | **~3.0-4.0G** | 🔥 |

## 🚨 When NOT to Use This Buildpack

- ❌ **Web frontends** - Use standard buildpack with serve
- ❌ **Full-stack apps** - Too aggressive, might break assets
- ❌ **Apps needing Node.js** - Optimized for Bun runtime
- ❌ **Apps with many runtime deps** - Better to bundle more

## ✅ Best Practices

1. **Bundle as much as possible** in your build script
   - Only externalize Prisma, database drivers, and Pothos core
   - Bundle graphql-yoga, envelop, and other pure JS packages

2. **Pre-generate Prisma client locally**
   - Commit `prisma/generated/` to version control
   - Faster builds, more reliable deployments

3. **Minimize external dependencies**
   - Each external package adds to node_modules size
   - Aim for <10 external packages in production

4. **Monitor your slug size**
   - Check build logs for final slug size
   - Aim to stay under 300M for safety margin

## 📚 Example Project Structure

```
apis/servers/api-client-server/
├── src/
│   ├── index.ts              # Server entry point
│   ├── builder.ts            # GraphQL schema
│   └── scripts/
│       └── build-heroku.ts   # Build script for this buildpack
├── prisma/
│   ├── schema/               # Prisma schema files
│   └── generated/            # ✅ Committed to Git
│       ├── client/           # Prisma client
│       └── crud/             # Generated CRUD resolvers
├── public/                   # Static assets
└── package.json
    {
      "scripts": {
        "build:heroku": "bun ./src/scripts/build-heroku.ts",
        "start": "bun ./dist/index.js"
      },
      "dependencies": {
        "@prisma/client": "catalog:",
        "mssql": "catalog:"
      }
    }
```

## 🤝 Contributing

This buildpack is part of the NewsoftDS monorepo. To make changes:

1. Edit files in `utils/buildpack-api/`
2. Test locally with a sample deployment
3. Update this README if adding features
4. Commit and push to trigger Heroku rebuild

---

**Built for NewsoftDS API servers** 🏗️ **Optimized for production** 🔥

# Task 10: Add Build Process and Optimization

## Priority: Low 📝

## Description
Implement a modern build process with bundling, minification, optimization, and development tools to improve performance and developer experience.

## Current Issue
The application is a single HTML file without any build process, missing opportunities for:
- Code optimization and minification
- Module bundling
- Asset optimization
- Development tools (hot reload, etc.)
- Performance optimization

## Solution
Implement a comprehensive build system using Vite or Webpack:

### Project Structure
```
project/
├── src/
│   ├── index.html
│   ├── main.ts
│   ├── components/
│   ├── services/
│   ├── styles/
│   └── assets/
├── dist/
├── package.json
├── vite.config.ts
└── tsconfig.json
```

### Package.json
```json
{
  "name": "voice-to-text-app",
  "version": "1.0.0",
  "description": "Modern voice-to-text application with speech recognition",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "lint": "eslint src --ext ts,js",
    "lint:fix": "eslint src --ext ts,js --fix",
    "format": "prettier --write src/**/*.{ts,js,css,html}",
    "type-check": "tsc --noEmit",
    "test": "vitest",
    "test:ui": "vitest --ui",
    "analyze": "npx vite-bundle-analyzer dist/assets/*.js"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "eslint": "^8.45.0",
    "prettier": "^3.0.0",
    "typescript": "^5.0.0",
    "vite": "^4.4.0",
    "vite-bundle-analyzer": "^0.7.0",
    "vitest": "^0.34.0"
  },
  "dependencies": {
    "workbox-precaching": "^7.0.0",
    "workbox-routing": "^7.0.0",
    "workbox-strategies": "^7.0.0"
  }
}
```

### Vite Configuration
```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import { resolve } from 'path';

export default defineConfig({
  root: 'src',
  build: {
    outDir: '../dist',
    emptyOutDir: true,
    minify: 'terser',
    target: 'es2020',
    rollupOptions: {
      input: {
        main: resolve(__dirname, 'src/index.html')
      },
      output: {
        manualChunks: {
          vendor: ['workbox-precaching', 'workbox-routing', 'workbox-strategies']
        }
      }
    },
    terserOptions: {
      compress: {
        drop_console: true,
        drop_debugger: true
      }
    }
  },
  server: {
    port: 3000,
    host: true,
    https: true // Required for speech recognition
  },
  optimizeDeps: {
    include: []
  },
  define: {
    __BUILD_TIME__: JSON.stringify(new Date().toISOString()),
    __VERSION__: JSON.stringify(process.env.npm_package_version || '1.0.0')
  }
});
```

### ESLint Configuration
```javascript
// .eslintrc.cjs
module.exports = {
  root: true,
  env: {
    browser: true,
    es2020: true
  },
  extends: [
    'eslint:recommended',
    '@typescript-eslint/recommended'
  ],
  parser: '@typescript-eslint/parser',
  parserOptions: {
    ecmaVersion: 'latest',
    sourceType: 'module'
  },
  plugins: ['@typescript-eslint'],
  rules: {
    '@typescript-eslint/no-unused-vars': ['error', { 'argsIgnorePattern': '^_' }],
    '@typescript-eslint/no-explicit-any': 'warn',
    'no-console': 'warn',
    'no-debugger': 'error',
    'prefer-const': 'error',
    'no-var': 'error'
  }
};
```

### Prettier Configuration
```json
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 80,
  "tabWidth": 2,
  "useTabs": false
}
```

### Service Worker for PWA
```typescript
// src/sw.ts
import { precacheAndRoute } from 'workbox-precaching';
import { registerRoute } from 'workbox-routing';
import { CacheFirst, StaleWhileRevalidate } from 'workbox-strategies';

// Precache all static assets
precacheAndRoute(self.__WB_MANIFEST);

// Cache audio API responses
registerRoute(
  /^https:\/\/.*\.googleapis\.com\/.*$/,
  new CacheFirst({
    cacheName: 'google-apis'
  })
);

// Cache other external resources
registerRoute(
  /^https:\/\/fonts\.googleapis\.com/,
  new StaleWhileRevalidate({
    cacheName: 'google-fonts-stylesheets'
  })
);

registerRoute(
  /^https:\/\/fonts\.gstatic\.com/,
  new CacheFirst({
    cacheName: 'google-fonts-webfonts',
    plugins: [{
      cacheKeyWillBeUsed: async ({ request }) => {
        return `${request.url}?v=1`;
      }
    }]
  })
);
```

### Environment Configuration
```typescript
// src/config/environment.ts
interface Config {
  isDevelopment: boolean;
  isProduction: boolean;
  version: string;
  buildTime: string;
  apiEndpoint?: string;
  enableAnalytics: boolean;
}

const config: Config = {
  isDevelopment: import.meta.env.DEV,
  isProduction: import.meta.env.PROD,
  version: __VERSION__,
  buildTime: __BUILD_TIME__,
  apiEndpoint: import.meta.env.VITE_API_ENDPOINT,
  enableAnalytics: import.meta.env.PROD && !import.meta.env.VITE_DISABLE_ANALYTICS
};

export default config;
```

### Performance Optimizations
```typescript
// src/utils/performance.ts
export class PerformanceMonitor {
  private static instance: PerformanceMonitor;
  private metrics: Map<string, number> = new Map();

  public static getInstance(): PerformanceMonitor {
    if (!PerformanceMonitor.instance) {
      PerformanceMonitor.instance = new PerformanceMonitor();
    }
    return PerformanceMonitor.instance;
  }

  public markStart(name: string): void {
    performance.mark(`${name}-start`);
  }

  public markEnd(name: string): void {
    performance.mark(`${name}-end`);
    performance.measure(name, `${name}-start`, `${name}-end`);
    
    const measure = performance.getEntriesByName(name)[0];
    this.metrics.set(name, measure.duration);
  }

  public getMetrics(): Record<string, number> {
    return Object.fromEntries(this.metrics);
  }

  public reportVitals(): void {
    // Report Core Web Vitals
    import('web-vitals').then(({ getCLS, getFID, getFCP, getLCP, getTTFB }) => {
      getCLS(console.log);
      getFID(console.log);
      getFCP(console.log);
      getLCP(console.log);
      getTTFB(console.log);
    });
  }
}

// Code splitting utility
export const loadModule = async <T>(
  moduleFactory: () => Promise<T>
): Promise<T> => {
  try {
    return await moduleFactory();
  } catch (error) {
    console.error('Module loading failed:', error);
    throw error;
  }
};
```

### Bundle Analysis
```typescript
// scripts/analyze-bundle.ts
import { analyzeBundle } from 'vite-bundle-analyzer';

async function analyze() {
  const analysis = await analyzeBundle({
    bundlePath: './dist',
    reportFilename: 'bundle-report.html'
  });
  
  console.log('Bundle analysis complete:');
  console.log(`Total size: ${(analysis.totalSize / 1024 / 1024).toFixed(2)}MB`);
  console.log(`Gzipped: ${(analysis.gzippedSize / 1024 / 1024).toFixed(2)}MB`);
  
  // Warn about large bundles
  if (analysis.totalSize > 1024 * 1024) { // 1MB
    console.warn('⚠️  Bundle size exceeds 1MB');
  }
  
  // Show largest modules
  console.log('\nLargest modules:');
  analysis.modules
    .sort((a, b) => b.size - a.size)
    .slice(0, 10)
    .forEach(module => {
      console.log(`${module.name}: ${(module.size / 1024).toFixed(2)}KB`);
    });
}

analyze().catch(console.error);
```

### Development Scripts
```bash
#!/bin/bash
# scripts/dev.sh
echo "Starting development server..."

# Check for HTTPS certificates
if [ ! -f "localhost.pem" ]; then
  echo "Generating HTTPS certificates for speech recognition..."
  mkcert localhost 127.0.0.1 ::1
fi

# Start development server
npm run dev
```

### Production Build Optimization
```typescript
// vite.config.production.ts
import { defineConfig } from 'vite';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  // ... base config
  plugins: [
    visualizer({
      filename: 'dist/stats.html',
      open: true,
      gzipSize: true
    })
  ],
  build: {
    // ... existing build config
    rollupOptions: {
      output: {
        manualChunks: {
          'speech-recognition': ['./src/services/SpeechRecognitionService'],
          'state-management': ['./src/state/AppStateManager'],
          'ui-components': ['./src/ui/UIController']
        }
      }
    }
  }
});
```

## Acceptance Criteria
- [ ] Modern build system with Vite/Webpack
- [ ] TypeScript compilation and bundling
- [ ] Code minification and optimization
- [ ] Development server with hot reload
- [ ] Production build optimization
- [ ] Bundle analysis and size monitoring
- [ ] Linting and formatting tools
- [ ] Performance monitoring
- [ ] PWA capabilities with service worker

## Files to Create
- `package.json` with dependencies and scripts
- `vite.config.ts` for build configuration
- `.eslintrc.cjs` for code linting
- `.prettierrc` for code formatting
- Service worker for PWA features
- Performance monitoring utilities
- Build analysis scripts

## Build Targets
- **Development**: Fast builds, source maps, hot reload
- **Production**: Minified, optimized, tree-shaken
- **Bundle size**: Target < 500KB gzipped
- **Performance**: First Contentful Paint < 1.5s

## Testing
1. Development build works with hot reload
2. Production build is optimized and minified
3. Bundle analysis shows reasonable sizes
4. Linting and formatting work correctly
5. Performance metrics are collected
6. PWA features work offline

## Estimated Time
3-4 hours
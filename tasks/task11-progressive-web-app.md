# Task 11: Implement Progressive Web App Features

## Priority: Low 📝

## Description
Transform the voice-to-text application into a Progressive Web App (PWA) with offline support, installability, push notifications, and native app-like experience.

## Current Issue
The application is a basic web page without PWA capabilities:
- No offline functionality
- Not installable
- No native app features
- No background sync
- No push notifications

## Solution
Implement comprehensive PWA features:

### Web App Manifest
```json
// public/manifest.json
{
  "name": "Voice to Text Converter",
  "short_name": "VoiceText",
  "description": "Convert speech to text using advanced speech recognition",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#1a1a2e",
  "theme_color": "#4f46e5",
  "orientation": "portrait-primary",
  "categories": ["productivity", "utilities", "accessibility"],
  "lang": "en",
  "dir": "ltr",
  "icons": [
    {
      "src": "/icons/icon-72x72.png",
      "sizes": "72x72",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/icons/icon-96x96.png",
      "sizes": "96x96",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/icons/icon-128x128.png",
      "sizes": "128x128",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/icons/icon-144x144.png",
      "sizes": "144x144",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/icons/icon-152x152.png",
      "sizes": "152x152",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/icons/icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/icons/icon-384x384.png",
      "sizes": "384x384",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/icons/icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "maskable any"
    }
  ],
  "shortcuts": [
    {
      "name": "Start Recording",
      "short_name": "Record",
      "description": "Quickly start voice recording",
      "url": "/?action=record",
      "icons": [
        {
          "src": "/icons/record-shortcut.png",
          "sizes": "192x192"
        }
      ]
    },
    {
      "name": "View Transcripts",
      "short_name": "Transcripts",
      "description": "View saved transcripts",
      "url": "/?action=transcripts",
      "icons": [
        {
          "src": "/icons/transcripts-shortcut.png",
          "sizes": "192x192"
        }
      ]
    }
  ],
  "screenshots": [
    {
      "src": "/screenshots/desktop-home.png",
      "sizes": "1280x720",
      "type": "image/png",
      "form_factor": "wide",
      "label": "Voice to Text home screen on desktop"
    },
    {
      "src": "/screenshots/mobile-recording.png",
      "sizes": "375x812",
      "type": "image/png",
      "form_factor": "narrow",
      "label": "Recording voice on mobile"
    }
  ]
}
```

### Enhanced Service Worker
```typescript
// src/sw.ts
import { precacheAndRoute, cleanupOutdatedCaches } from 'workbox-precaching';
import { registerRoute, NavigationRoute } from 'workbox-routing';
import { CacheFirst, NetworkFirst, StaleWhileRevalidate } from 'workbox-strategies';
import { BackgroundSync } from 'workbox-background-sync';
import { Queue } from 'workbox-background-sync';

// Precache static assets
precacheAndRoute(self.__WB_MANIFEST);
cleanupOutdatedCaches();

// Cache strategies for different resource types
registerRoute(
  ({ request }) => request.destination === 'document',
  new NetworkFirst({
    cacheName: 'pages',
    networkTimeoutSeconds: 3
  })
);

registerRoute(
  ({ request }) => 
    request.destination === 'style' ||
    request.destination === 'script' ||
    request.destination === 'worker',
  new StaleWhileRevalidate({
    cacheName: 'assets'
  })
);

registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({
    cacheName: 'images',
    plugins: [{
      cacheKeyWillBeUsed: async ({ request }) => {
        return `${request.url}?v=1`;
      }
    }]
  })
);

// Background sync for saving transcripts
const bgSyncQueue = new Queue('transcript-sync', {
  onSync: async ({ queue }) => {
    let entry;
    while ((entry = await queue.shiftRequest())) {
      try {
        await fetch(entry.request.clone());
        console.log('Transcript synced successfully');
      } catch (error) {
        console.error('Failed to sync transcript:', error);
        throw error;
      }
    }
  }
});

// Listen for transcript save requests
self.addEventListener('message', (event) => {
  if (event.data && event.data.type === 'SAVE_TRANSCRIPT') {
    const { transcript, timestamp } = event.data.payload;
    
    // Save to IndexedDB for offline access
    saveTranscriptOffline(transcript, timestamp);
    
    // Queue for background sync
    bgSyncQueue.pushRequest({
      request: new Request('/api/transcripts', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ transcript, timestamp })
      })
    });
  }
});

// Offline transcript storage
async function saveTranscriptOffline(transcript: string, timestamp: number) {
  const db = await openTranscriptDB();
  const transaction = db.transaction(['transcripts'], 'readwrite');
  const store = transaction.objectStore('transcripts');
  
  await store.add({
    id: `transcript_${timestamp}`,
    content: transcript,
    timestamp,
    synced: false
  });
}

// IndexedDB setup
async function openTranscriptDB(): Promise<IDBDatabase> {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open('VoiceTextDB', 1);
    
    request.onerror = () => reject(request.error);
    request.onsuccess = () => resolve(request.result);
    
    request.onupgradeneeded = (event) => {
      const db = (event.target as IDBOpenDBRequest).result;
      
      if (!db.objectStoreNames.contains('transcripts')) {
        const store = db.createObjectStore('transcripts', { keyPath: 'id' });
        store.createIndex('timestamp', 'timestamp', { unique: false });
        store.createIndex('synced', 'synced', { unique: false });
      }
    };
  });
}

// Push notification handling
self.addEventListener('push', (event) => {
  if (!event.data) return;
  
  const data = event.data.json();
  const options = {
    body: data.body,
    icon: '/icons/icon-192x192.png',
    badge: '/icons/badge-72x72.png',
    vibrate: [100, 50, 100],
    data: data,
    actions: [
      {
        action: 'open',
        title: 'Open App',
        icon: '/icons/open-action.png'
      },
      {
        action: 'dismiss',
        title: 'Dismiss',
        icon: '/icons/dismiss-action.png'
      }
    ]
  };
  
  event.waitUntil(
    self.registration.showNotification(data.title, options)
  );
});

// Handle notification clicks
self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  
  if (event.action === 'open' || !event.action) {
    event.waitUntil(
      clients.matchAll({ type: 'window' }).then((clientList) => {
        // Focus existing window or open new one
        for (const client of clientList) {
          if (client.url.includes('/') && 'focus' in client) {
            return client.focus();
          }
        }
        if (clients.openWindow) {
          return clients.openWindow('/');
        }
      })
    );
  }
});
```

### PWA Installation Handler
```typescript
// src/services/PWAService.ts
export class PWAService {
  private deferredPrompt: any = null;
  private isInstalled = false;

  constructor() {
    this.setupInstallPrompt();
    this.detectInstallation();
  }

  private setupInstallPrompt(): void {
    window.addEventListener('beforeinstallprompt', (event) => {
      event.preventDefault();
      this.deferredPrompt = event;
      this.showInstallButton();
    });
  }

  private detectInstallation(): void {
    // Check if running as PWA
    if (window.matchMedia('(display-mode: standalone)').matches ||
        (window.navigator as any).standalone === true) {
      this.isInstalled = true;
      this.hideInstallButton();
    }

    // Listen for installation
    window.addEventListener('appinstalled', () => {
      this.isInstalled = true;
      this.hideInstallButton();
      this.trackInstallation();
    });
  }

  public async installApp(): Promise<void> {
    if (!this.deferredPrompt) {
      throw new Error('Install prompt not available');
    }

    const result = await this.deferredPrompt.prompt();
    
    if (result.outcome === 'accepted') {
      console.log('PWA installation accepted');
    } else {
      console.log('PWA installation dismissed');
    }

    this.deferredPrompt = null;
  }

  public isAppInstalled(): boolean {
    return this.isInstalled;
  }

  private showInstallButton(): void {
    const installButton = document.getElementById('install-app-btn');
    if (installButton) {
      installButton.style.display = 'block';
      installButton.addEventListener('click', () => this.installApp());
    }
  }

  private hideInstallButton(): void {
    const installButton = document.getElementById('install-app-btn');
    if (installButton) {
      installButton.style.display = 'none';
    }
  }

  private trackInstallation(): void {
    // Analytics tracking
    if (typeof gtag !== 'undefined') {
      gtag('event', 'pwa_install', {
        event_category: 'PWA',
        event_label: 'Installation'
      });
    }
  }
}
```

### Offline Storage Service
```typescript
// src/services/OfflineStorageService.ts
export class OfflineStorageService {
  private db: IDBDatabase | null = null;
  private dbName = 'VoiceTextDB';
  private version = 1;

  async initialize(): Promise<void> {
    this.db = await this.openDatabase();
  }

  private openDatabase(): Promise<IDBDatabase> {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open(this.dbName, this.version);
      
      request.onerror = () => reject(request.error);
      request.onsuccess = () => resolve(request.result);
      
      request.onupgradeneeded = (event) => {
        const db = (event.target as IDBOpenDBRequest).result;
        
        // Transcripts store
        if (!db.objectStoreNames.contains('transcripts')) {
          const transcriptStore = db.createObjectStore('transcripts', { 
            keyPath: 'id', 
            autoIncrement: true 
          });
          transcriptStore.createIndex('timestamp', 'timestamp', { unique: false });
          transcriptStore.createIndex('language', 'language', { unique: false });
          transcriptStore.createIndex('synced', 'synced', { unique: false });
        }
        
        // Settings store
        if (!db.objectStoreNames.contains('settings')) {
          const settingsStore = db.createObjectStore('settings', { keyPath: 'key' });
        }
      };
    });
  }

  async saveTranscript(transcript: TranscriptData): Promise<number> {
    if (!this.db) throw new Error('Database not initialized');
    
    const transaction = this.db.transaction(['transcripts'], 'readwrite');
    const store = transaction.objectStore('transcripts');
    
    const transcriptWithMeta = {
      ...transcript,
      timestamp: Date.now(),
      synced: navigator.onLine
    };
    
    return new Promise((resolve, reject) => {
      const request = store.add(transcriptWithMeta);
      request.onsuccess = () => resolve(request.result as number);
      request.onerror = () => reject(request.error);
    });
  }

  async getTranscripts(limit = 50): Promise<TranscriptData[]> {
    if (!this.db) throw new Error('Database not initialized');
    
    const transaction = this.db.transaction(['transcripts'], 'readonly');
    const store = transaction.objectStore('transcripts');
    const index = store.index('timestamp');
    
    return new Promise((resolve, reject) => {
      const request = index.openCursor(null, 'prev');
      const results: TranscriptData[] = [];
      let count = 0;
      
      request.onsuccess = (event) => {
        const cursor = (event.target as IDBRequest).result;
        if (cursor && count < limit) {
          results.push(cursor.value);
          count++;
          cursor.continue();
        } else {
          resolve(results);
        }
      };
      
      request.onerror = () => reject(request.error);
    });
  }

  async deleteSyncedTranscripts(): Promise<void> {
    if (!this.db) throw new Error('Database not initialized');
    
    const transaction = this.db.transaction(['transcripts'], 'readwrite');
    const store = transaction.objectStore('transcripts');
    const index = store.index('synced');
    
    return new Promise((resolve, reject) => {
      const request = index.openCursor(IDBKeyRange.only(true));
      
      request.onsuccess = (event) => {
        const cursor = (event.target as IDBRequest).result;
        if (cursor) {
          cursor.delete();
          cursor.continue();
        } else {
          resolve();
        }
      };
      
      request.onerror = () => reject(request.error);
    });
  }
}

interface TranscriptData {
  id?: number;
  content: string;
  language: string;
  timestamp?: number;
  synced?: boolean;
}
```

### Push Notification Service
```typescript
// src/services/PushNotificationService.ts
export class PushNotificationService {
  private vapidPublicKey = 'YOUR_VAPID_PUBLIC_KEY';
  
  async requestPermission(): Promise<NotificationPermission> {
    if ('Notification' in window) {
      return Notification.requestPermission();
    }
    return 'denied';
  }

  async subscribe(): Promise<PushSubscription | null> {
    if ('serviceWorker' in navigator && 'PushManager' in window) {
      const registration = await navigator.serviceWorker.ready;
      
      try {
        const subscription = await registration.pushManager.subscribe({
          userVisibleOnly: true,
          applicationServerKey: this.urlB64ToUint8Array(this.vapidPublicKey)
        });
        
        // Send subscription to server
        await this.sendSubscriptionToServer(subscription);
        return subscription;
      } catch (error) {
        console.error('Failed to subscribe to push notifications:', error);
        return null;
      }
    }
    return null;
  }

  private urlB64ToUint8Array(base64String: string): Uint8Array {
    const padding = '='.repeat((4 - base64String.length % 4) % 4);
    const base64 = (base64String + padding)
      .replace(/-/g, '+')
      .replace(/_/g, '/');

    const rawData = window.atob(base64);
    const outputArray = new Uint8Array(rawData.length);

    for (let i = 0; i < rawData.length; ++i) {
      outputArray[i] = rawData.charCodeAt(i);
    }
    return outputArray;
  }

  private async sendSubscriptionToServer(subscription: PushSubscription): Promise<void> {
    await fetch('/api/push-subscribe', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(subscription)
    });
  }
}
```

## Acceptance Criteria
- [ ] Web App Manifest with proper metadata
- [ ] Service Worker with offline caching
- [ ] App installation prompt and handling
- [ ] Offline transcript storage with IndexedDB
- [ ] Background sync for data synchronization
- [ ] Push notification support
- [ ] App shortcuts and share target
- [ ] Native app-like experience
- [ ] Performance meets PWA standards

## Files to Create/Modify
- `public/manifest.json` - Web App Manifest
- Enhanced service worker with offline support
- PWA installation service
- Offline storage service with IndexedDB
- Push notification service
- App icons and screenshots
- PWA-specific UI components

## PWA Requirements Checklist
- [ ] HTTPS deployment
- [ ] Web App Manifest
- [ ] Service Worker
- [ ] Offline functionality
- [ ] Responsive design
- [ ] Fast loading (< 3s)
- [ ] App-like interactions

## Testing
1. Lighthouse PWA audit (score > 90)
2. Test offline functionality
3. Verify app installation works
4. Test background sync
5. Validate push notifications
6. Check responsive design
7. Test app shortcuts

## Estimated Time
4-6 hours
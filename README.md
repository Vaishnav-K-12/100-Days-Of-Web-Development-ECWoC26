/* A service worker for 100 Days of Web */

const VERSION = 'v1';
const CORE_CACHE = `core-${VERSION}`;
const RUNTIME_CACHE = `runtime-${VERSION}`;

const CORE_ASSETS = [
  'index.html',

  'website/style.css',
  'website/theme.js',
  'website/script.js',
  'website/tracker.js',

  'website/projects.html',
  'website/about.html',
  'website/contributors.html',
  'website/contact.html',
  'website/login.html',
  'website/contribute.html',


  'website/community.json',

  'manifest.webmanifest',
  'pwa-icon.svg'
];

self.addEventListener('install', (event) => {
  self.skipWaiting();
  event.waitUntil(
    caches.open(CORE_CACHE).then((cache) =>
      cache.addAll(CORE_ASSETS.map((url) => new Request(url, { cache: 'reload' })))
    )
  );
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    (async () => {
      const keys = await caches.keys();
      await Promise.all(
        keys
          .filter((key) => key.startsWith('core-') || key.startsWith('runtime-'))
          .filter((key) => key !== CORE_CACHE && key !== RUNTIME_CACHE)
          .map((key) => caches.delete(key))
      );

      await self.clients.claim();
    })()
  );
});

function isHtmlNavigationRequest(request) {
  if (request.mode === 'navigate') return true;
  const accept = request.headers.get('accept') || '';
  return accept.includes('text/html');
}

function shouldCacheStaticAsset(pathname) {
  if (pathname === '/manifest.webmanifest' || pathname === '/pwa-icon.svg') return true;
  if (pathname === '/' || pathname === '/index.html') return true;
  return pathname.startsWith('/website/');
}

self.addEventListener('fetch', (event) => {
  const { request } = event;

  if (request.method !== 'GET') return;

  const url = new URL(request.url);
  if (url.origin !== self.location.origin) return;

  const scopePath = new URL(self.registration.scope).pathname;
  const normalizedPathname = url.pathname.startsWith(scopePath)
    ? url.pathname.slice(scopePath.length - 1)
    : url.pathname;

  // HTML: network-first, cache for offline access later.
  if (isHtmlNavigationRequest(request)) {
    event.respondWith(
      (async () => {
        // Define a canonical URL for index.html (relative to the service worker's scope).
        // This ensures consistent caching for both '/index.html' and the root path '/'.
        const canonicalIndexUrl = new URL('index.html', self.registration.scope).toString();
        // Determine if the current request is for the root path or index.html based on normalized pathname.
        const isRootOrIndexRequest = (normalizedPathname === '/' || normalizedPathname === '/index.html');

        try {
          const response = await fetch(request);
          const cache = await caches.open(RUNTIME_CACHE);
          // If it's a root or index request, cache it under the canonical index.html URL
          // to prevent duplicate entries for '/' and '/index.html'.
          if (isRootOrIndexRequest) {
            cache.put(canonicalIndexUrl, response.clone());
          } else {
            cache.put(request, response.clone());
          }
          return response;
        } catch {
          let cachedResponse;
          // 1. Try to get the exact requested page from cache.
          cachedResponse = await caches.match(request);
          if (cachedResponse) return cachedResponse;

          // 2. If the exact page isn't found, try the canonical index.html URL.
          //    This covers cases where '/' was requested but 'index.html' was cached,
          //    or vice-versa, and also serves as a general fallback to the home page
          //    if any other navigation request fails.
          cachedResponse = await caches.match(canonicalIndexUrl);
          if (cachedResponse) return cachedResponse;

          // 3. If all attempts to retrieve from cache fail, return the offline message.
          return new Response('Offline: content not available yet.', {
            status: 503,
            headers: { 'Content-Type': 'text/plain; charset=utf-8' }
          });
        }
      })()
    );
    return;
  }

  // Assets: cache-first for known core/static paths.
  if (shouldCacheStaticAsset(normalizedPathname)) {
    event.respondWith(
      (async () => {
        const cached = await caches.match(request);
        if (cached) return cached;

        try {
          const response = await fetch(request);
          const cache = await caches.open(RUNTIME_CACHE);
          cache.put(request, response.clone());
          return response;
        } catch {
          return new Response('', { status: 504 });
        }
      })()
    );
  }
});
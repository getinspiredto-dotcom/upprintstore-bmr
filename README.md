// Service Worker — UpprintStore BMR Tracker
// Version du cache — incrémenter pour forcer une mise à jour
const CACHE_NAME = 'upprintstore-bmr-v1';

// Fichiers à mettre en cache pour le mode hors ligne
const ASSETS = [
  './',
  './index.html',
  './manifest.json',
  './icons/icon-192.png',
  './icons/icon-512.png',
  // Chart.js depuis CDN — mis en cache au premier chargement
  'https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.0/chart.umd.min.js',
  // Google Fonts
  'https://fonts.googleapis.com/css2?family=DM+Sans:wght@300;400;500;600;700&family=DM+Mono:wght@400;500&display=swap'
];

// ── INSTALLATION ──────────────────────────────────────────────────────────
self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => {
        console.log('[SW] Mise en cache des ressources...');
        // Mettre en cache les ressources locales d'abord (les CDN peuvent échouer)
        return cache.addAll(['./', './index.html', './manifest.json'])
          .then(() => {
            // Essayer les ressources CDN (peut échouer hors ligne)
            return Promise.allSettled(
              ASSETS.slice(3).map(url => 
                cache.add(url).catch(e => console.warn('[SW] Impossible de cacher:', url))
              )
            );
          });
      })
      .then(() => self.skipWaiting())
  );
});

// ── ACTIVATION ────────────────────────────────────────────────────────────
self.addEventListener('activate', event => {
  event.waitUntil(
    caches.keys()
      .then(keys => Promise.all(
        keys
          .filter(key => key !== CACHE_NAME)
          .map(key => {
            console.log('[SW] Suppression ancien cache:', key);
            return caches.delete(key);
          })
      ))
      .then(() => self.clients.claim())
  );
});

// ── FETCH ─────────────────────────────────────────────────────────────────
self.addEventListener('fetch', event => {
  // Stratégie : Cache First → Network Fallback
  event.respondWith(
    caches.match(event.request)
      .then(cached => {
        if (cached) return cached;
        
        // Pas en cache → fetch depuis le réseau
        return fetch(event.request)
          .then(response => {
            // Ne mettre en cache que les réponses valides
            if (!response || response.status !== 200 || response.type === 'opaque') {
              return response;
            }
            // Mettre en cache pour la prochaine fois
            const toCache = response.clone();
            caches.open(CACHE_NAME).then(cache => cache.put(event.request, toCache));
            return response;
          })
          .catch(() => {
            // Hors ligne et pas en cache → page d'erreur basique
            if (event.request.destination === 'document') {
              return caches.match('./index.html');
            }
          });
      })
  );
});

// ── MESSAGE ───────────────────────────────────────────────────────────────
self.addEventListener('message', event => {
  if (event.data === 'skipWaiting') self.skipWaiting();
});

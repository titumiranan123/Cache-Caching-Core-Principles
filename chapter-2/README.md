# Chapter 2: Cache এর প্রকারভেদ (Types of Caching)

একটা web request যখন browser থেকে শুরু হয়ে server পর্যন্ত যায়, তার পথে কোথায় কোথায় cache বসতে পারে সেটাই এই chapter এর বিষয়।

## পুরো Journey টা

```
[Browser Cache] → [CDN Cache] → [Load Balancer] → [Reverse Proxy Cache (Nginx)] 
    → [Application Cache (Redis/Memcached)] → [Database Query Cache] → [Disk]
```

## ১. Browser Cache (Client-Side Cache)

সবচেয়ে কাছের cache — ইউজারের নিজের browser এ।

**কী কী cache হয়:**
- Static files: CSS, JS, Images, Fonts
- API responses (কখনো কখনো)

**কীভাবে কাজ করে:**
Server response এর সাথে HTTP headers পাঠায় (`Cache-Control`, `ETag`) যেগুলো browser কে বলে দেয় কতক্ষণ data রাখতে হবে।

```
Cache-Control: max-age=31536000  (এক বছর cache থাকবে)
```

**Real-world উদাহরণ:** সাইট প্রথমবার visit করলে CSS/JS/logo ডাউনলোড হয়। দ্বিতীয়বার visit করলে এগুলো আর ডাউনলোড হয় না — browser নিজের disk থেকে load করে।

## ২. CDN Cache (Content Delivery Network)

**সমস্যা:** Server আমেরিকায়, ইউজার বাংলাদেশে — data অনেক দূর থেকে আসতে হয় → latency বেশি।

**সমাধান:** CDN (Cloudflare, AWS CloudFront, Akamai) সারা পৃথিবীর বিভিন্ন জায়গায় **edge server** রাখে, যেখানে static content এর copy cache করা থাকে।

```
User (Dhaka) → Nearest Edge Server (Singapore/Mumbai) → Cached Content ⚡
```

**কী cache হয়:** Images, videos, CSS, JS, এমনকি পুরো HTML page।

**Production Impact:** Netflix, Facebook, YouTube — এদের ভিডিও/ছবি সরাসরি origin server থেকে আসে না, CDN থেকে আসে।

## ৩. Reverse Proxy / Web Server Cache (Nginx, Varnish)

Application server এর সামনে reverse proxy (Nginx) বসিয়ে পুরো HTTP response cache করে রাখা যায়।

```nginx
location / {
    proxy_cache my_cache;
    proxy_cache_valid 200 10m;
}
```

## ৪. Application-Level Cache (In-Memory)

**a) Local/In-Process Cache**
- Application এর মেমোরিতে সরাসরি রাখা
- সমস্যা: একাধিক server instance এ প্রতিটার cache আলাদা → inconsistent data

```javascript
const cache = {};
function getUser(id) {
  if (cache[id]) return cache[id]; // Cache hit
  const user = database.query(id); // Cache miss
  cache[id] = user;
  return user;
}
```

**b) Distributed/Shared Cache (Redis, Memcached)**
- সব server instance central cache server এর সাথে connect থাকে
- সব সার্ভার একই data দেখে → consistency ঠিক থাকে

```
Server 1 ─┐
Server 2 ─┼──→ Redis (Shared Cache) ──→ Database
Server 3 ─┘
```

## ৫. Database-Level Cache

- **Query Cache**: একই query বার বার চললে result cache রাখে
- **Buffer Pool/Page Cache**: Database data disk থেকে RAM এ রাখে

## ৬. Full Production Flow উদাহরণ

```
1. User browser → check Browser Cache → না থাকলে →
2. CDN (Cloudflare) → static assets থাকলে সরাসরি দেয় →
3. Request Nginx এ আসে → page cache আছে কিনা check →
4. না থাকলে → Application Server (Node.js/Django) →
5. Redis check করে (product data cache আছে?) →
6. Redis এ না থাকলে → Database query →
7. Database result → Redis এ save (পরের বার এর জন্য) → 
8. Response ইউজারকে ফেরত
```

## ৭. কোন Cache কখন ব্যবহার করবেন?

| Data Type | কোথায় Cache করবেন |
|---|---|
| Images, CSS, JS, Fonts | Browser + CDN |
| Static pages (blog, landing page) | CDN + Reverse Proxy |
| User-specific data (profile, cart) | Application Cache (Redis) |
| Frequently read, rarely changed data | Redis |
| Session data | Redis |
| Real-time leaderboard, counters | Redis (special data structures) |
| Heavy computed reports | Application Cache with TTL |

---

## ✅ Summary
- Cache শুধু একটা জায়গায় না, পুরো request path জুড়ে multiple layer এ থাকে
- Browser Cache → CDN → Reverse Proxy → Application Cache (Redis) → Database Cache
- Redis (Distributed Application Cache) backend engineer হিসেবে সবচেয়ে বেশি নিয়ন্ত্রিত হয়
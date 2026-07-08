# Chapter 5: Browser Caching (HTTP Cache-Control, ETag, Last-Modified)

## ১. Browser Caching কীভাবে কাজ করে

```
Browser ──Request──→ Server
Browser ←─Response + Cache Headers──── Server

পরবর্তী Request এ:
Browser ── "আমার কাছে cache আছে, লাগবে কি?" ──→ Server
```

## ২. `Cache-Control` Header

```http
Cache-Control: max-age=3600, public
```

| Directive | মানে |
|---|---|
| `max-age=3600` | ৩৬০০ সেকেন্ড পর্যন্ত cache valid থাকবে |
| `public` | Browser এবং CDN দুটোই cache করতে পারবে |
| `private` | শুধু browser cache করবে |
| `no-cache` | Cache রাখবে কিন্তু ব্যবহারের আগে server কে জিজ্ঞেস করবে |
| `no-store` | একদমই cache করবে না (sensitive data) |
| `must-revalidate` | Expire হয়ে গেলে অবশ্যই server এ গিয়ে চেক করতে হবে |
| `immutable` | কখনো পরিবর্তন হবে না |

```javascript
// Static assets
app.use('/static', express.static('public', {
  maxAge: '1y',
  immutable: true
}));

// Sensitive data
app.get('/api/bank-balance', (req, res) => {
  res.set('Cache-Control', 'no-store');
  res.json(balance);
});

// Semi-dynamic API
app.get('/api/products', (req, res) => {
  res.set('Cache-Control', 'public, max-age=300'); // ৫ মিনিট
  res.json(products);
});
```

## ৩. Filename Versioning

```html
<!-- পুরনো (bad) -->
<link rel="stylesheet" href="style.css">

<!-- Production approach -->
<link rel="stylesheet" href="style.a3f5e9.css">
```

Content বদলালে filename এর hash বদলায় → browser নতুন resource মনে করে ডাউনলোড করে। এই জন্য React/Vue/Next.js build এ ফাইলের নামে random hash যুক্ত হয়।

## ৪. ETag (Entity Tag)

```http
# প্রথম Response
HTTP/1.1 200 OK
ETag: "5d8c72a5edda"
Cache-Control: max-age=3600
```

```http
# Cache expire হওয়ার পর
GET /style.css
If-None-Match: "5d8c72a5edda"

# ফাইল বদলায়নি:
HTTP/1.1 304 Not Modified

# বদলে গিয়ে থাকলে:
HTTP/1.1 200 OK
ETag: "new-hash-here"
[নতুন content]
```

`304 Not Modified` এ কোনো body থাকে না, তাই bandwidth বাঁচে।

## ৫. Last-Modified / If-Modified-Since

```http
Last-Modified: Wed, 01 Jul 2026 10:00:00 GMT
If-Modified-Since: Wed, 01 Jul 2026 10:00:00 GMT
HTTP/1.1 304 Not Modified
```

ETag বেশি accurate (content hash based), Last-Modified কম accurate (timestamp based)।

## ৬. পুরো Flow

```
প্রথমবার Page Load:
Browser → Request → Server
Server → 200 OK + Cache-Control + ETag
Browser → Cache তে save

দ্বিতীয়বার (১ ঘণ্টার আগে):
Browser → Cache থেকেই নিলো, Server এ request-ই গেলো না ⚡

দ্বিতীয়বার (expired):
Browser → Server কে জিজ্ঞেস করলো (If-None-Match)
বদলায়নি → 304 Not Modified
বদলেছে → 200 OK + নতুন data
```

## ৭. Production Best Practices

**Static Assets (hash সহ filename):**
```http
Cache-Control: public, max-age=31536000, immutable
```

**HTML Pages:**
```http
Cache-Control: no-cache
```

**API Responses:**
```http
Cache-Control: private, max-age=60
```

**Sensitive Data:**
```http
Cache-Control: no-store
```

## ৮. Common Mistake

```
no-cache   → Cache রাখো, কিন্তু ব্যবহারের আগে জিজ্ঞেস করো
no-store   → একদমই cache করো না
```

---

## ✅ Summary
- `Cache-Control: max-age` দিয়ে কতক্ষণ cache থাকবে ঠিক হয়
- **ETag** দিয়ে content-based validation (`304 Not Modified`)
- Static assets এ filename hashing + দীর্ঘ `max-age`
- HTML সবসময় `no-cache` রাখা উচিত
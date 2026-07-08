# Chapter 3: Cache Eviction Policies (LRU, LFU, FIFO, TTL)

## ১. সমস্যাটা কী?

Cache (RAM) এর জায়গা limited। Database এ হয়তো ১০০ GB ডেটা আছে, কিন্তু Redis server এ হয়তো মাত্র ২ GB RAM বরাদ্দ।

প্রশ্ন হলো: **যখন cache পুরো ভরে যাবে, নতুন ডেটা রাখার জন্য কোন পুরনো ডেটা ফেলে দেবেন?**

এই সিদ্ধান্ত নেওয়ার নিয়মকেই বলে **Eviction Policy**।

## ২. TTL (Time To Live)

প্রতিটা cache entry এর একটা expiry time সেট করে দেওয়া হয়। সময় শেষ হলে সেই data নিজে থেকেই মুছে যায়।

```javascript
redis.set("product:101", JSON.stringify(product), "EX", 300); // 300 seconds = 5 min
```

**Production Rule of Thumb:**
- খুব দ্রুত পরিবর্তনশীল data (stock price, live score) → TTL কম (৫-৩০ সেকেন্ড)
- মোটামুটি স্থির data (product info, blog post) → TTL বেশি (৫-৬০ মিনিট)
- প্রায় স্থির data (country list, config) → TTL অনেক বেশি (ঘণ্টা/দিন)

## ৩. LRU (Least Recently Used)

**Logic:** যে ডেটা সবচেয়ে বেশি সময় ধরে ব্যবহার হয়নি, সেটা প্রথমে বাদ দাও।

```
Cache (max 3 items), LRU policy:

Access: A, B, C          → Cache: [A, B, C]
Access: A                → Cache: [B, C, A]
নতুন D আসলো              → Cache ফুল, B evict (সবচেয়ে বেশিদিন অব্যবহৃত)
Cache: [C, A, D]
```

**Redis এ:** `maxmemory-policy allkeys-lru`

**কোথায় ব্যবহার হয়:** সবচেয়ে বেশি ব্যবহৃত default policy। Facebook, Twitter timeline cache, user session cache।

## ৪. LFU (Least Frequently Used)

**Logic:** যে ডেটা সবচেয়ে কম সংখ্যকবার access হয়েছে, সেটা বাদ দাও।

```
Item A: accessed 100 বার
Item B: accessed 2 বার
Item C: accessed 50 বার

Memory ফুল হলে → B প্রথমে evict হবে
```

**LRU vs LFU:** একটা popular item ৫ মিনিট ধরে access না হলেও LRU সেটাকে evict করতে পারে, কিন্তু LFU total access count দেখে বলবে "এটা popular, রাখো"।

**Redis এ:** `maxmemory-policy allkeys-lfu`

## ৫. FIFO (First In First Out)

**Logic:** যে data সবার আগে ঢুকেছে, সেটাই সবার আগে বের হবে, ব্যবহার হয়েছে কিনা তা বিবেচনা করা হয় না।

```
Enter order: A, B, C, D
Memory ফুল → A বের হবে (সবার আগে ঢুকেছিল)
```

কম smart, তাই production এ কম ব্যবহার হয়।

## ৬. Redis এর সব Eviction Policy

| Policy | ব্যাখ্যা |
|---|---|
| `noeviction` | কিছুই evict করবে না, নতুন write এ error দেবে (Default) |
| `allkeys-lru` | সব key থেকে LRU অনুযায়ী evict করে |
| `volatile-lru` | শুধু TTL সেট করা key গুলোর মধ্যে LRU evict করে |
| `allkeys-lfu` | সব key থেকে LFU অনুযায়ী evict করে |
| `volatile-lfu` | শুধু TTL সেট করা key গুলোর মধ্যে LFU evict করে |
| `allkeys-random` | Random ভাবে যেকোনো key evict করে |
| `volatile-random` | TTL সেট করা key থেকে random evict করে |
| `volatile-ttl` | যেটার TTL সবচেয়ে কম বাকি আছে, সেটা আগে evict করে |

```bash
maxmemory 2gb
maxmemory-policy allkeys-lru
```

## ৭. Real Production Scenario

**Case 1: E-commerce product cache** → `allkeys-lru` + TTL ৫-১০ মিনিট

**Case 2: Session storage** → `volatile-ttl` অথবা শুধু TTL (LRU দিয়ে ফেলে দিলে ইউজার হঠাৎ logout হয়ে যাবে)

**Case 3: Trending/Viral content** → `allkeys-lfu`

## ৮. Interview Question

**প্রশ্ন:** TTL আর Eviction Policy কি একই জিনিস?

**উত্তর:** না।
- **TTL** = ডেটার নির্দিষ্ট expiry time, সময় শেষ হলে memory থাকুক বা না থাকুক মুছে যায়
- **Eviction Policy** = memory ফুল হয়ে গেলে কোন ডেটা জোর করে বের করে দেওয়া হবে তার নিয়ম

---

## ✅ Summary
- **TTL**: সময়ভিত্তিক expiry — সবচেয়ে common ও দরকারি
- **LRU**: সাম্প্রতিক ব্যবহার অনুযায়ী evict — সবচেয়ে বহুল ব্যবহৃত policy
- **LFU**: কতবার ব্যবহার হয়েছে তা অনুযায়ী evict
- **FIFO**: সহজ কিন্তু কম smart
# Chapter 4: Cache Invalidation Strategies (সবচেয়ে কঠিন সমস্যা)

## ১. কেন এটা এত কঠিন?

Cache এর সবচেয়ে বড় বিপদ হলো **Stale Data**।

**উদাহরণ:** প্রোডাক্টের দাম ১০০০ টাকা, Redis এ cache করা ৫ মিনিটের TTL দিয়ে। Admin দাম কমিয়ে ৮০০ টাকা করলেন database এ। কিন্তু cache জানে না! পরবর্তী ৫ মিনিট ধরে ইউজাররা ভুল দাম দেখতে থাকবে।

```
Database: price = 800 (updated)
Redis Cache: price = 1000 (stale/পুরনো) ❌
```

## ২. প্রধান তিনটি Invalidation Strategy

### Strategy 1: TTL-based (Passive Expiration)

```javascript
redis.set("product:101", data, "EX", 300); // 5 মিনিট পর নিজে থেকে মুছে যাবে
```

**সুবিধা:** Simple। **অসুবিধা:** ওই সময় ধরে stale data দেখানো হবে।

### Strategy 2: Write-Through Invalidation (Active/Manual)

Database update হওয়ার সাথে সাথেই সরাসরি cache delete করে দেওয়া।

```javascript
async function updateProductPrice(productId, newPrice) {
  await db.query("UPDATE products SET price = ? WHERE id = ?", [newPrice, productId]);
  await redis.del(`product:${productId}`);
}
```

**এটাই production এ সবচেয়ে বেশি ব্যবহৃত** critical data (price, inventory) এর জন্য।

### Strategy 3: Event-Driven Invalidation (Pub/Sub)

```javascript
// Service A
await redis.publish("product-updated", JSON.stringify({ id: 101 }));

// Service B
redis.subscribe("product-updated", (message) => {
  const { id } = JSON.parse(message);
  redis.del(`product:${id}`);
});
```

## ৩. Cache-Aside (Lazy Loading) Pattern

```
Read করার সময়:
1. প্রথমে Cache চেক করো
2. থাকলে (Hit) → রিটার্ন করো
3. না থাকলে (Miss) → Database থেকে আনো → Cache এ রাখো → রিটার্ন করো

Write করার সময়:
1. Database update করো
2. Cache থেকে সেই key delete করো
```

```javascript
async function getProduct(id) {
  let product = await redis.get(`product:${id}`);
  if (product) return JSON.parse(product);
  
  product = await db.query("SELECT * FROM products WHERE id = ?", [id]);
  await redis.set(`product:${id}`, JSON.stringify(product), "EX", 300);
  return product;
}
```

## ৪. Thundering Herd / Cache Stampede

**সমস্যা:** Popular product এর cache expire হয়ে গেল, ঠিক তখনই ১০,০০০ ইউজার একসাথে request করলো → সবাই একসাথে database এ হিট করে → Database crash!

**সমাধান — Locking:**

```javascript
async function getProduct(id) {
  let product = await redis.get(`product:${id}`);
  if (product) return JSON.parse(product);

  const lockAcquired = await redis.set(`lock:${id}`, "1", "NX", "EX", 10);
  
  if (lockAcquired) {
    product = await db.query("SELECT * FROM products WHERE id = ?", [id]);
    await redis.set(`product:${id}`, JSON.stringify(product), "EX", 300);
    await redis.del(`lock:${id}`);
    return product;
  } else {
    await sleep(100);
    return getProduct(id); // retry
  }
}
```

**আরেকটা সমাধান:** Stale-While-Revalidate — পুরনো data সাথে সাথে দেখিয়ে দাও, background এ নতুন data fetch করে cache update করো।

## ৫. Real Production Example

```javascript
async function placeOrder(productId, quantity) {
  const stock = await redis.get(`stock:${productId}`);
  if (stock < quantity) throw new Error("Out of stock");
  
  await db.transaction(async (trx) => {
    await trx.query("UPDATE inventory SET stock = stock - ? WHERE product_id = ?", 
      [quantity, productId]);
    await trx.query("INSERT INTO orders ...");
  });
  
  await redis.del(`stock:${productId}`);
}
```

## ৬. Decision Table

| Data Type | Strategy |
|---|---|
| দাম, স্টক (accuracy জরুরি) | Write-Through Invalidation |
| User profile info | Write-Through |
| "Related products" | TTL-based |
| Multi-service data | Event-driven (Pub/Sub) |
| খুব popular item | TTL + Locking |

---

## ✅ Summary
- তিনটা মূল approach: **TTL-based**, **Write-Through**, **Event-driven**
- **Thundering Herd** সমস্যা — locking বা stale-while-revalidate দিয়ে সমাধান
- Production এ প্রায় সবসময় **TTL + Manual invalidation** একসাথে ব্যবহার হয়
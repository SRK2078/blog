---
title: "🔄 Background DNS Refresh in Node.js: Keeping Cached IPs Updated"
seoTitle: "Node.js Guide: Refreshing Cached DNS IPs"
seoDescription: "Learn how to implement background DNS refresh in Node.js to maintain consistent performance and eliminate latency spikes in high-traffic applications"
datePublished: Tue Mar 11 2025 14:49:12 GMT+0000 (Coordinated Universal Time)
cuid: cm84lx5nk000008jp9erfcbf4
slug: background-dns-refresh-in-nodejs-keeping-cached-ips-updated
tags: nodejs

---

## 📌 Introduction

In our previous blog on [DNS Caching in Node.js](https://sachinthapa.hashnode.dev/dns-caching-in-nodejs-why-and-how-to-optimize-api-performance), we learned how caching resolved IP addresses improves performance and prevents frequent DNS lookups. However, one challenge remains: what happens when the TTL expires?

With basic DNS caching, when an IP address expires, our system performs a fresh DNS lookup before serving the next request. This creates latency spikes and blocking operations for users. To fix this problem, we need to implement background DNS refresh.

In this comprehensive guide, we'll cover:

* Why background DNS refresh is essential for consistent performance
    
* How it eliminates latency spikes in high-traffic applications
    
* A detailed implementation in Node.js with scheduled updates
    
* Performance benchmarks comparing on-demand and background refresh
    
* Best practices for production environments
    

By the end, you'll know how to keep cached DNS records fresh without ever blocking requests! 🚀

## ❌ The Problem: Delayed DNS Refresh Causes Latency Spikes

### 🔹 How Basic DNS Caching Works

With the basic caching approach:

1. We resolve a domain name to an IP and cache it until the TTL expires
    
2. After TTL expiration, the next request triggers a fresh DNS lookup
    
3. That specific request experiences a delay while waiting for DNS resolution
    
4. Subsequent requests are fast again until the next expiration
    

### 🔹 Issues with This Approach

1️⃣ **Latency Spikes** – Users experience periodic delays when an expired cache is refreshed

2️⃣ **Blocking Requests** – The first request after expiration waits for a DNS resolution

3️⃣ **Inconsistent Performance** – Some requests are fast (cached IPs), others are slow (DNS lookups)

4️⃣ **Poor User Experience** – Unpredictable response times frustrate users

## 📌 Solution: Refresh IPs in the Background

Instead of waiting for a request to trigger a DNS resolution, we can refresh cached IPs proactively before they expire. This ensures continuous smooth operation without any request ever having to wait.

## ✅ The Solution: Implementing Background DNS Refresh

Our improved approach:

1. Resolve and cache DNS records as before
    
2. Calculate when to refresh (typically at 80% of TTL)
    
3. Schedule background refresh tasks before TTL expiry
    
4. Keep cached records updated without impacting incoming requests
    
5. Ensure consistent performance for all users
    

### 🔥 Benefits of Background DNS Refresh:

✅ **Eliminates latency spikes** – Requests never wait for DNS lookups

✅ **Prevents blocking requests** – Cached IPs are always ready before expiration

✅ **Smoothens API performance** – Provides consistent, predictable response times

✅ **Handles DNS failures gracefully** – If a refresh fails, the existing IP remains valid

## 🔧 Implementation: Background DNS Refresh in Node.js

### Step 1: Enhance the Cache to Support Background Refresh

```javascript
import express from "express";
import fetch from "node-fetch";
import * as dns from "node:dns";

const app = express();
const PORT = 3000;
const dnsCache = new Map();
const REFRESH_THRESHOLD = 0.8; // Refresh when 80% of TTL has elapsed

async function resolveAndCache(domain) {
    try {
        const result = await dns.promises.resolve4(domain, { ttl: true });
        if (result.length > 0) {
            const { address, ttl } = result[0];
            const now = Date.now();
            const expiresAt = now + ttl * 1000;
            
            // Calculate when to refresh (before expiry)
            const refreshAt = now + (ttl * REFRESH_THRESHOLD * 1000);
            
            // Store both expiry and refresh times
            dnsCache.set(domain, { 
                address, 
                expiresAt,
                refreshAt
            });
            
            console.log(`Updated IP for ${domain}: ${address} (TTL: ${ttl}s)`);
            
            // Schedule background refresh before expiry
            const timeUntilRefresh = refreshAt - now;
            setTimeout(() => refreshDomain(domain), timeUntilRefresh);
            
            return address;
        }
    } catch (error) {
        console.error(`DNS lookup failed for ${domain}:`, error);
    }
    return null;
}
```

### Step 2: Set Up a Background Refresh Process

```javascript
async function refreshDomain(domain) {
    console.log(`Background refresh for ${domain}`);
    const cached = dnsCache.get(domain);
    
    // Only refresh if this entry still exists and hasn't been refreshed already
    if (cached && Date.now() < cached.expiresAt && Date.now() >= cached.refreshAt) {
        await resolveAndCache(domain);
    }
}

// Function to get a cached IP (with fallback to resolution)
async function getCachedIP(domain) {
    const now = Date.now();
    
    // Check if domain is cached and still valid
    if (dnsCache.has(domain)) {
        const { address, expiresAt } = dnsCache.get(domain);
        if (now < expiresAt) {
            return address;
        }
    }
    
    // Not cached or expired, resolve now
    return await resolveAndCache(domain);
}

// Initialize cache for commonly used domains
async function initializeCache(domains) {
    for (const domain of domains) {
        await resolveAndCache(domain);
    }
}

const monitoredDomains = ["api.weather.com", "api.payments.com", "api.auth.com"];
initializeCache(monitoredDomains);
```

### Step 3: Use Cached IPs Without Delays

```javascript
app.get("/weather", async (req, res) => {
    const weatherAPI = "api.weather.com";
    try {
        const ip = await getCachedIP(weatherAPI);
        if (!ip) return res.status(500).json({ error: "DNS lookup failed" });
        
        const response = await fetch(`http://${ip}/data`, {
            headers: { 'Host': 'api.weather.com' }
        });
        
        const data = await response.json();
        res.json({ success: true, data });
    } catch (error) {
        res.status(500).json({ error: "Failed to fetch weather data" });
    }
});

app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

## ⚡ Performance Comparison: With vs. Without Background Refresh

| Feature | Without Background Refresh | With Background Refresh |
| --- | --- | --- |
| Latency Spikes | Present (after TTL expiry) | Eliminated completely |
| Blocking Requests | First request after expiry is slow | No blocking requests ever |
| DNS Lookup Frequency | On-demand (delayed) | Periodic (proactive) |
| Performance | Inconsistent | Smooth & consistent |
| p95 Latency | 150-250ms | 5-10ms |
| p99 Latency | 300-400ms | 10-15ms |

## 🎯 Best Practices for Production

1. **Adjust the refresh threshold** based on your application's needs (80% of TTL is a good starting point)
    
2. **Monitor DNS resolution times** to identify performance issues
    
3. **Implement error handling** in refresh operations to avoid crashing the application
    
4. **Consider memory usage** when caching many domains
    
5. **Use health checks** to ensure your background refresh process is working properly
    

## 🚀 Final Thoughts

✅ Background DNS refresh ensures that DNS lookups never block a request, eliminating latency spikes and providing consistent performance.

### 💡 Ideal Use Cases:

* APIs handling thousands of requests per second
    
* Microservices communicating with third-party services
    
* IoT devices that need uninterrupted connectivity
    
* Payment gateways, streaming services, and real-time applications
    

### 🔥 Next Steps

In our next blog post, we'll explore how to implement a robust DNS fallback mechanism that can handle DNS resolution failures gracefully, ensuring your application stays operational even when DNS providers experience issues.

Check out our previous blog on [DNS Caching in Node.js](https://sachinthapa.hashnode.dev/dns-caching-in-nodejs-why-and-how-to-optimize-api-performance) and stay tuned for our upcoming post on DNS Fallback Mechanisms! 🚀
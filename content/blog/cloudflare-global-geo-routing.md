---
date: '2025-12-21T16:09:22+03:00'
draft: false
title: 'How I built Global Geo-Routing for Free on Cloudflare (Saving $300/Year)'
tags: ["Cloudflare", "CloudflareWorkers", "EdgeComputing", "Serverless", "JavaScript", "DevOps"]
---

## The Problem: Global latency on a Budget

I recently took on a project for a client with a classic distributed architecture problem. They had a great application running on Docker containers, frontend by Traefik, deployed across four distinct region:

- Germany (EU Central)
- United States (US East)
- Australia (Oceania)
- South Africa (Africa)

**The Goal**: A single domain name (like `app.example.com`). When a user visits the domain, they must be routed to geographically closest server. A user in Berlin should hit the Germany server; a user in Sydney should hit the Australia server.

## The "Standard" (Expensive) Solution

If you look through Cloudflare's documentation for this, they point you directly to their **Load Balancing** product with the **Geo-Steering** add-on.

It is an excellent product that works flawlessly. But let's look at the [pricing](https://www.cloudflare.com/plans/) for a small-to-medium project (at the writing time):

- Load Balancing Subscription: ~$5/mo
- Geo-Steering Feature: ~$10/mo
- Additional Origins & Health Checks: +$$

You are quickly looking at **$20-$30 per month**, or upwards of **$360 per year**. For a startup or a lean project, paying recurrently just to direct traffic based on locations feels excessive.

I knew there had to be a smarter way to solve this without the enterprise price tag.

## The "Smart" Solution: Cloudflare Workers

I realized that the core requirement wasn't complex load balancing algorithms or sophisticated health checks; it was simple **if/then logic based on geography**.

Cloudflare [Workers](https://developers.cloudflare.com/workers/) allows you to run JavaScript at the nework edge, before a request hits your server. Crutially, every incoming request to Cloudflare includes a header containing the visitor's two-letter country code: `request.cf.country`.

By combining a Worker with Cloudflare's generous [free tier](https://developers.cloudflare.com/workers/platform/pricing/) (100,000 requests/day), I could build a custom geo-router for **$0**.

### Implementation (The Code)

The architecture is simple. The DNS for the main domain points to Cloudflare. A Worker sits on that route, intercepts the request, checks the country header, and proxies the request to the correct regional backend hostname.

Here is the core logic of the script I deployed:

```JavaScript
export default {
  async fetch(request, env, ctx) {
    // 1. Get the visitor's country from Cloudflare's magic header
    const country = request.cf.country;
    const url = new URL(request.url);

    // 2. Define your regional backends map
    // Map specific countries to their nearest server hub
    const backends = {
      // Europe goes to Germany
      "DE": "de-server.example-backend.com",
      "FR": "de-server.example-backend.com",
      "GB": "de-server.backend.com",
      
      // Oceania goes to Australia
      "AU": "au-server.example-backend.com",
      "NZ": "au-server.example-backend.com",
      
      // Africa goes to South Africa
      "ZA": "za-server.example-backend.com",
      // ... add more countries as needed
    };

    // 3. Select the Target
    // Use the country map, or fall back to the US server for unlisted regions
    const targetHost = backends[country] || "us-server.example-backend.com";

    // 4. Point the request to the new target hostname
    url.hostname = targetHost;

    // 5. Create the new request to send to the backend
    // Important: Pass the original request info along
    const newRequest = new Request(url, request);

    // --- THE CRITICAL PART (See below) ---
    // We tell the backend app what the ORIGINAL domain was.
    newRequest.headers.set("X-Forwarded-Host", "app.example.com");
    newRequest.headers.set("X-Forwarded-Proto", "https");

    // 6. Execute the proxy fetch
    return fetch(newRequest);
  },
};
```

#### The "Gotcha": Host headers and SSL Errors:

It wasn't smooth sailing immediately. My first attempt simply changed the `url.hostname` and fetched.

**The result? A 404 or SSL handshake error.**

Why? When the worker reached out to the backend server (e.g., `de-server.example-backend.com`), it was using the original domain (`app.example.com`) in the connection handshake `Host` header. The backed server's SSL certificate didn't recognize that domain, so it rejected the connection.

**The Fix:** We have to let Cloudflare manage the connection handshake naturally using the backend's actual hostname. Instead of overriding the `Host` header, we use standard porxy headers to tell the backend application who the *real* visitor is.

```JavaScript
// Don't touch the 'Host' header. Use these instead:
newRequest.headers.set("X-Forwarded-Host", "app.example.com");
newRequest.headers.set("X-Forwarded-Proto", "https");
```

This ensured the SSL handshake succeeded, and the Traefik proxy on the backend knew exactly which site to serve.

### Verification and Results

To prove to the client that it was working, I added a temporary debug header to the Worker's reponse just before returning it to the user:

```JavaScript
// Add a header showing which server handled the request
const response = await fetch(newRequest);
const newResponse = new Response(response.body, response);
newResponse.headers.set("X-Debug-Backend", targetHost);
return newResponse;
```

Using a global ping like KeyCDN, we could test from different cities around the world and inspect the responce headers. A request from Berlin showed `X-Debug-Backend: de-server...`, While a request from Cape Town showed `X-Debug-Backend: za-server...`.

**Success. Total latency was drastically reduced for global users, and total cost remained $0.**

## Conclusion

Enterprise features are fantastic when you need enterprise scale. But for many use cases, a little bit of knowledge of edge computing and Javascript can save you significant recurring costs.

By using Cloudflare Workers dynamically, I saved the client over $300 a year and delivered a robust, highly performant global routing solution in under 50 lines of codes.


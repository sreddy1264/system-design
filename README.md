# System Design

A collection of high-level system design documents covering real-world distributed systems — architecture diagrams, component breakdowns, data flows, key design decisions, and scale numbers.

---

## Reference

| Topic | Description |
|-------|-------------|
| [System Design Trade-offs](systemDesign/tradeoffs.md) | SQL vs NoSQL, vertical vs horizontal scaling, consistency vs availability, caching, fan-out, and 15 other core trade-offs |

---

## Designs

| System | Description |
|--------|-------------|
| [Amazon](systemDesign/amazon.md) | E-commerce platform — product catalog, inventory, orders, fulfillment, and ML-driven recommendations |
| [Dropbox](systemDesign/dropbox.md) | Cloud file storage — chunked upload, delta sync, content deduplication, and real-time multi-device sync |
| [Instagram](systemDesign/instagram.md) | Photo and video sharing — feed generation, media processing, stories, and social graph |
| [Netflix](systemDesign/netflix.md) | Video streaming — content delivery, adaptive bitrate, encoding pipeline, and personalized recommendations |
| [Notification System](systemDesign/notificationSystem.md) | Multi-channel notification platform — push (APNs/FCM), email, SMS, in-app, priority lanes, and retry/DLQ |
| [Uber](systemDesign/uber.md) | Ride-hailing platform — real-time location tracking, driver matching, surge pricing, and trip lifecycle |
| [URL Shortener](systemDesign/urlShortner.md) | URL shortening service — Base62 encoding, cache-first redirects, Bloom filter, and click analytics |
| [WhatsApp](systemDesign/whatsapp.md) | Messaging platform — end-to-end encryption, message delivery guarantees, and group messaging |
| [X (Twitter)](systemDesign/x.md) | Social media platform — fan-out on write/read hybrid, real-time search, timeline ranking, and trending topics |
| [YouTube](systemDesign/youtube.md) | Video hosting platform — upload pipeline, transcoding, CDN delivery, and view count aggregation |


# Day 7 Assignment: Scaling SnapShare

## 1. Assumptions

SnapShare is a photo-sharing application where users upload photos and view feeds containing photos from people they follow.

The calculations use the following assumptions from the assignment:

- Registered users: 10,000,000
- Daily active users: 10% of registered users
- Uploads per active user: 1 photo per day
- Feed views per active user: 50 per day
- Average original photo size: 2 MB
- Average thumbnail size: 50 KB
- Seconds per day: 86,400
- Peak traffic: 5 times the average traffic
- Year: 365 days

For storage calculations, 1 MB = 1,000 KB and 1 TB = 1,000,000 MB. The estimates exclude database records, backups, replicas, and other overhead.

## 2. Traffic and Storage Estimates

### Daily active users

10,000,000 registered users × 10%

Daily active users = **1,000,000**

### Uploads per second

Each active user uploads one photo per day.

Uploads per day = 1,000,000 × 1 = 1,000,000

Average uploads per second:

1,000,000 ÷ 86,400 ≈ **11.6 uploads/second**

Peak uploads per second:

11.6 × 5 ≈ **58 uploads/second**

### Feed views per second

Each active user views 50 feed pages per day.

Feed views per day = 1,000,000 × 50 = 50,000,000

Average feed views per second:

50,000,000 ÷ 86,400 ≈ **579 feed views/second**

Peak feed views per second:

579 × 5 ≈ **2,895 feed views/second**

### Photo storage per year

Each uploaded photo requires storage for the original and its thumbnail.

Thumbnail size = 50 KB ÷ 1,000 = 0.05 MB

Total storage per photo = 2 MB + 0.05 MB = 2.05 MB

Daily storage:

1,000,000 photos × 2.05 MB = 2,050,000 MB = **2.05 TB/day**

Yearly storage:

2.05 TB × 365 = **748.25 TB/year**

Breakdown:

- Original photos: 2 TB/day × 365 = 730 TB/year.
- Thumbnails: 0.05 TB/day × 365 = 18.25 TB/year.
- Total: 748.25 TB/year.

This is the additional storage required for one year's uploads, before accounting for backups, replication, and storage overhead.

### Summary of estimates

| Measurement | Average | Peak (5×) |
|---|---:|---:|
| Daily active users | 1,000,000 | — |
| Photo uploads/second | 11.6 | 58 |
| Feed views/second | 579 | 2,895 |
| New photo storage/year | 748.25 TB | — |

## 3. Is SnapShare Read-Heavy or Write-Heavy?

SnapShare is **read-heavy** because users view approximately 50 million feed pages per day, compared with one million photo uploads per day.

There are approximately 50 feed views for every photo upload.

This means the system should optimise feed loading using caching, a CDN, read replicas, and multiple application servers. Uploads also need to be reliable, but most requests are expected to be reads rather than uploads.

## 4. Where Should Photos Be Stored?

Photo files should be stored in object storage, such as Amazon S3 or a compatible object storage service, rather than directly inside the relational database.

Photo files are large, and storing them in object storage makes it easier to scale storage independently, serve images through a CDN, and manage thumbnails.

The database should store metadata such as the photo ID, owner ID, caption, upload time, and object storage key or URL. This keeps database records smaller and makes them easier to query.

## 5. Proposed Architecture Diagram

```text
                    ┌──────────────┐
                    │    Users     │
                    │ Web / Mobile │
                    └──────┬───────┘
                           │
                           v
                    ┌──────────────┐
                    │     DNS      │
                    └──────┬───────┘
                           │
             ┌─────────────┴─────────────┐
             │                           │
             v                           v
      ┌──────────────┐           ┌────────────────┐
      │ CDN          │           │ Load Balancer  │
      │ Photos,      │           └───────┬────────┘
      │ Thumbnails   │                   │
      └──────┬───────┘             ┌─────┴─────┐
             │                     v           v
             │               ┌─────────┐  ┌─────────┐
             │               │ App     │  │ App     │
             │               │ Server 1│  │ Server 2│
             │               └────┬────┘  └────┬────┘
             │                    │            │
             │                    └─────┬──────┘
             │                          │
             │               ┌──────────┴─────────┐
             │               │                    │
             │               v                    v
             │        ┌──────────────┐     ┌──────────────┐
             │        │ Cache        │     │ Primary DB   │
             │        │ Redis        │     │ Metadata     │
             │        └──────────────┘     └──────┬───────┘
             │                                    │
             │                                    v
             │                             ┌──────────────┐
             │                             │ Read Replica │
             │                             └──────────────┘
             │
             v
      ┌──────────────────┐
      │ Object Storage   │
      │ Original Photos  │
      │ and Thumbnails   │
      └──────────────────┘

      Upload processing:

      App Server ──> Object Storage (original photo)
           │
           └──────> Message Queue ──> Thumbnail Worker
                                            │
                                            v
                                      Object Storage
                                      (thumbnail)
```

## 6. What Each Component Does

- **DNS:** Translates SnapShare's domain name into the address needed to reach its services.
- **CDN:** Serves photos and thumbnails from nearby edge locations, reducing image loading time and origin traffic.
- **Load balancer:** Distributes incoming API requests across healthy application servers.
- **Application servers:** Authenticate users, validate uploads, manage feeds, and coordinate database and storage operations.
- **Cache (Redis):** Keeps frequently requested feed data or metadata in memory to reduce database load.
- **Primary database:** Stores authoritative photo metadata, user information, relationships, and other structured records.
- **Read replica:** Serves eligible read queries so the primary database has more capacity for writes.
- **Object storage:** Stores large original photo files and generated thumbnails durably.
- **Message queue:** Holds thumbnail-generation jobs so photo uploads do not have to wait for image processing to finish.
- **Thumbnail worker:** Processes queued jobs, generates smaller image versions, and saves them to object storage.

The application servers should be stateless: important user and photo data belongs in durable storage, not only in a server's memory.

## 7. Step-by-Step Photo Upload Flow

1. A user selects a photo and submits it through the SnapShare application.
2. The application server authenticates the user and validates the upload, including the file type, size, and applicable limits.
3. The original photo is uploaded to object storage. A secure upload mechanism, such as a short-lived pre-signed URL, can allow the client to upload directly to storage.
4. The server records the photo metadata in the primary database, including the owner, storage key, caption, and upload status.
5. The server adds a thumbnail-generation job to the message queue.
6. The application responds that the upload has been accepted or completed, depending on the storage and metadata operations that have succeeded.
7. A background worker retrieves the job and generates a smaller thumbnail.
8. The worker saves the thumbnail to object storage and updates the photo's processing status in the database.
9. When users view their feeds, the application retrieves feed metadata from the cache or database and returns photo URLs.
10. The browser loads the original photos or thumbnails through the CDN, which can retrieve uncached files from object storage.

If thumbnail generation fails temporarily, the job can be retried without requiring the user to upload the photo again. The system should also handle partial failures so that an uploaded original is not treated as successfully processed before the required operations are complete.

## 8. Trade-Offs

### Trade-off 1: Speed versus freshness

Caching feeds reduces response time and database load, but cached feeds may briefly be outdated when someone uploads a new photo.

SnapShare can use short cache expiration times or invalidate affected cache entries when new posts are created.

### Trade-off 2: Cost versus reliability

Multiple application servers, read replicas, backups, and durable object storage improve reliability and capacity, but increase infrastructure costs.

SnapShare should start with a modest deployment and add capacity based on measured demand instead of building an unnecessarily large system from day one.

### Trade-off 3: Immediate processing versus background processing

Generating thumbnails during the upload request makes the user wait longer. Processing them through a queue and worker makes uploads faster, but thumbnails may not be available immediately.

SnapShare can show the original photo or a processing indicator until the thumbnail is ready.

### Trade-off 4: Database consistency versus read performance

Read replicas and caches can briefly return older data because updates take time to propagate.

SnapShare can use the primary database for operations requiring the latest data and use replicas or caches for feed reads where brief delays are acceptable.

## 9. Conclusion

SnapShare needs an architecture that supports many more feed views than uploads and can store hundreds of terabytes of new image data each year. A CDN, cache, load balancer, multiple stateless application servers, a primary database with a read replica, object storage, and background thumbnail workers help address these requirements.

The design should grow gradually as actual traffic and operational measurements reveal where more capacity is needed.
```

## 2. Check your work

Before committing, confirm that the file includes:

- [ ] Daily active users and upload calculations.
- [ ] Average and peak feed views per second.
- [ ] Annual photo and thumbnail storage estimate.
- [ ] All required architecture components.
- [ ] A sentence explaining each component.
- [ ] Upload flow, including the thumbnail worker.
- [ ] At least two trade-offs.

## 3. Commit and push to GitHub

From the root of `web-foundations-days`, run:

```bash
git add day7/photo-app-scaling.md
git commit -m "Day 7 assignment"
git push
```

Your Day 7 assignment is ready to test and submit. Remember that these figures are estimates based on the given assumptions, not a prediction of SnapShare's exact production traffic.
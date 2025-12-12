# Long-Running Download Architecture Design

## The Problem

Our microservice handles file downloads with highly variable processing times (10-120 seconds). When deployed behind reverse proxies like Cloudflare, nginx, or AWS ALB, this creates critical problems:

1. **Connection Timeouts**: Proxies terminate connections after 100 seconds
2. **Poor User Experience**: Users see no progress feedback during 2+ minute waits
3. **Resource Exhaustion**: Open HTTP connections consume server memory
4. **Retry Storms**: Dropped connections trigger duplicate requests

## Chosen Solution: Polling Pattern with Job Queue

After evaluating WebSocket, SSE, Webhook, and Hybrid approaches, we selected the **Polling Pattern** for these reasons:

**Why Polling Wins:**
- Works with all proxies and firewalls
- Simple client implementation
- No persistent connections required
- Easy to cache and scale horizontally
- Familiar pattern for developers
- Lower infrastructure cost than WebSockets

**Trade-offs Accepted:**
- Slightly higher latency (2-3 second polling interval)
- More HTTP requests than push-based solutions
- Acceptable because: jobs take 10-120s, polling overhead is negligible

## Architecture Diagram
```
┌──────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                              │
│                                                                   │
│  ┌─────────────┐         ┌──────────────┐                       │
│  │   Browser   │────────▶│  React App   │                       │
│  │             │         │  (useDownload│                       │
│  └─────────────┘         │    Hook)     │                       │
│                          └───────┬──────┘                        │
└──────────────────────────────────┼───────────────────────────────┘
                                   │
                  ┌────────────────┼────────────────┐
                  │                │                │
            1. POST /initiate      │          3. GET /status/:jobId
               (instant)           │             (every 2s)
                  │                │                │
                  ▼                │                ▼
┌──────────────────────────────────────────────────────────────────┐
│                    REVERSE PROXY LAYER                            │
│                 (Cloudflare / nginx / ALB)                        │
│                Timeout: 100s (not a problem)                      │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                  ┌─────────┴─────────┐
                  ▼                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API LAYER (Hono)                            │
│                                                                  │
│  POST /v1/download/initiate     GET /v1/download/status/:jobId  │
│  └─▶ Create job in queue        └─▶ Query Redis for status     │
│  └─▶ Return jobId (50ms)        └─▶ Return progress (10ms)     │
│                                                                  │
└────────────────┬─────────────────────────────────┬──────────────┘
                 │                                 │
                 │                                 │
        2. Queue job                      Fast status lookup
                 │                                 │
                 ▼                                 ▼
┌─────────────────────────────┐    ┌──────────────────────────────┐
│     JOB QUEUE (BullMQ)      │    │      REDIS CACHE             │
│                             │    │                              │
│  ┌──────────────────────┐   │    │  job:abc123 = {             │
│  │ Queue: downloads     │   │    │    status: "processing",    │
│  │ Priority: FIFO       │   │    │    progress: 2/5,           │
│  │ Concurrency: 5       │   │    │    files: [...]             │
│  └──────────────────────┘   │    │  }                          │
│                             │    │  TTL: 24 hours              │
└──────────────┬──────────────┘    └──────────────────────────────┘
               │                                 ▲
               │ Dequeue job                     │
               │                                 │ Update status
               ▼                                 │
┌─────────────────────────────────────────────────────────────────┐
│                   WORKER POOL (Node.js)                          │
│                                                                  │
│  Worker 1    Worker 2    Worker 3    Worker 4    Worker 5       │
│  [Processing][Processing][Idle]      [Processing][Idle]         │
│                                                                  │
│  For each file:                                                 │
│  1. Update status to "processing"                               │
│  2. Simulate download (10-120s)                                 │
│  3. Upload result to MinIO                                      │
│  4. Update status to "completed"                                │
│                                                                  │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            │ Upload completed files
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    STORAGE LAYER (MinIO)                         │
│                                                                  │
│  Bucket: downloads/                                             │
│  ├── job_abc123_file_70000.zip                                 │
│  ├── job_abc123_file_70001.zip                                 │
│  └── job_abc123_bundle.zip                                     │
│                                                                  │
│  Presigned URLs expire after 24 hours                           │
└─────────────────────────────────────────────────────────────────┘
```

## Technical Implementation

### 1. API Contract

#### New Endpoint: POST /v1/download/initiate

**Request:**
```json
{
  "file_ids": [70000, 70001, 70002]
}
```

**Response (Immediate ~50ms):**
```json
{
  "job_id": "job_abc123def",
  "status": "queued",
  "created_at": "2025-12-12T10:00:00Z",
  "total_files": 3,
  "estimated_completion": "2025-12-12T10:02:00Z"
}
```

#### New Endpoint: GET /v1/download/status/:jobId

**Response (Fast Redis lookup ~10ms):**
```json
{
  "job_id": "job_abc123def",
  "status": "processing",
  "progress": {
    "completed": 1,
    "total": 3,
    "percentage": 33
  },
  "files": [
    {
      "file_id": 70000,
      "status": "completed",
      "download_url": "https://minio.local/downloads/job_abc123_70000.zip",
      "expires_at": "2025-12-13T10:00:00Z"
    },
    {
      "file_id": 70001,
      "status": "processing",
      "started_at": "2025-12-12T10:00:30Z"
    },
    {
      "file_id": 70002,
      "status": "queued"
    }
  ],
  "updated_at": "2025-12-12T10:00:45Z"
}
```

**Final Response (when completed):**
```json
{
  "job_id": "job_abc123def",
  "status": "completed",
  "progress": {
    "completed": 3,
    "total": 3,
    "percentage": 100
  },
  "download_url": "https://minio.local/downloads/job_abc123_bundle.zip",
  "expires_at": "2025-12-13T10:00:00Z",
  "completed_at": "2025-12-12T10:02:15Z"
}
```

#### New Endpoint: DELETE /v1/download/:jobId

**Response:**
```json
{
  "job_id": "job_abc123def",
  "status": "cancelled",
  "message": "Job cancelled successfully"
}
```

### 2. Database Schema (Redis)

#### Job Record
```typescript
interface Job {
  job_id: string;
  status: 'queued' | 'processing' | 'completed' | 'failed' | 'cancelled';
  file_ids: number[];
  created_at: string;
  updated_at: string;
  completed_at?: string;
  progress: {
    completed: number;
    total: number;
    percentage: number;
  };
  bundle_url?: string;
  error?: string;
}

// Redis key: job:{job_id}
// TTL: 86400 seconds (24 hours)
```

#### File Status Record
```typescript
interface FileStatus {
  file_id: number;
  job_id: string;
  status: 'queued' | 'processing' | 'completed' | 'failed';
  started_at?: string;
  completed_at?: string;
  download_url?: string;
  size_bytes?: number;
  error?: string;
}

// Redis key: file:{job_id}:{file_id}
// TTL: 86400 seconds (24 hours)
```

### 3. Background Job Processing

**Technology Stack:**
- **Queue**: BullMQ (Redis-backed, reliable, scalable)
- **Workers**: Node.js processes (can scale independently)
- **Concurrency**: 5 jobs per worker

**Implementation:**
```typescript
import { Queue, Worker } from 'bullmq';
import Redis from 'ioredis';

const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: 6379,
  maxRetriesPerRequest: null
});

// Job Queue
const downloadQueue = new Queue('downloads', {
  connection: redis,
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 2000
    },
    removeOnComplete: {
      age: 86400,
      count: 1000
    },
    removeOnFail: {
      age: 86400
    }
  }
});

// Worker Process
const worker = new Worker('downloads', async (job) => {
  const { job_id, file_ids } = job.data;
  
  await updateJobStatus(job_id, 'processing');
  
  for (const file_id of file_ids) {
    await updateFileStatus(job_id, file_id, 'processing');
    
    try {
      const result = await processDownload(file_id);
      const url = await uploadToMinIO(result, job_id, file_id);
      
      await updateFileStatus(job_id, file_id, 'completed', url);
      await updateJobProgress(job_id);
      
    } catch (error) {
      await updateFileStatus(job_id, file_id, 'failed', null, error.message);
      throw error;
    }
  }
  
  const bundleUrl = await createBundle(job_id, file_ids);
  await updateJobStatus(job_id, 'completed', bundleUrl);
  
}, {
  connection: redis,
  concurrency: 5,
  limiter: {
    max: 10,
    duration: 1000
  }
});

// Helper Functions
async function updateJobStatus(jobId: string, status: string, bundleUrl?: string) {
  const key = `job:${jobId}`;
  const job = await redis.get(key);
  const jobData = JSON.parse(job);
  
  jobData.status = status;
  jobData.updated_at = new Date().toISOString();
  if (status === 'completed') {
    jobData.completed_at = new Date().toISOString();
    jobData.bundle_url = bundleUrl;
  }
  
  await redis.setex(key, 86400, JSON.stringify(jobData));
}

async function updateFileStatus(
  jobId: string,
  fileId: number,
  status: string,
  downloadUrl?: string,
  error?: string
) {
  const key = `file:${jobId}:${fileId}`;
  const data = {
    file_id: fileId,
    job_id: jobId,
    status,
    download_url: downloadUrl,
    error,
    updated_at: new Date().toISOString()
  };
  
  if (status === 'processing') {
    data.started_at = new Date().toISOString();
  } else if (status === 'completed') {
    data.completed_at = new Date().toISOString();
  }
  
  await redis.setex(key, 86400, JSON.stringify(data));
}

async function updateJobProgress(jobId: string) {
  const jobKey = `job:${jobId}`;
  const job = JSON.parse(await redis.get(jobKey));
  
  let completed = 0;
  for (const fileId of job.file_ids) {
    const fileKey = `file:${jobId}:${fileId}`;
    const file = JSON.parse(await redis.get(fileKey));
    if (file.status === 'completed') completed++;
  }
  
  job.progress = {
    completed,
    total: job.file_ids.length,
    percentage: Math.round((completed / job.file_ids.length) * 100)
  };
  
  await redis.setex(jobKey, 86400, JSON.stringify(job));
}
```

### 4. Error Handling & Retry Logic

**Retry Strategy:**
```typescript
const retryStrategy = {
  attempts: 3,
  backoff: {
    type: 'exponential',
    delay: 2000  // 2s, 4s, 8s
  }
};

enum ErrorType {
  NETWORK_ERROR = 'network_error',      // Retry ✓
  TIMEOUT_ERROR = 'timeout_error',      // Retry ✓
  STORAGE_ERROR = 'storage_error',      // Retry ✓
  INVALID_FILE = 'invalid_file',        // No retry ✗
  QUOTA_EXCEEDED = 'quota_exceeded'     // No retry ✗
}

async function processDownloadWithRetry(fileId: number, attempt: number = 1) {
  try {
    return await processDownload(fileId);
  } catch (error) {
    const errorType = classifyError(error);
    
    if (!shouldRetry(errorType) || attempt >= 3) {
      throw error;
    }
    
    const delay = Math.pow(2, attempt) * 1000;
    await sleep(delay);
    
    return processDownloadWithRetry(fileId, attempt + 1);
  }
}

function shouldRetry(errorType: ErrorType): boolean {
  return [
    ErrorType.NETWORK_ERROR,
    ErrorType.TIMEOUT_ERROR,
    ErrorType.STORAGE_ERROR
  ].includes(errorType);
}
```

**Dead Letter Queue:**
```typescript
const failedQueue = new Queue('downloads:failed', {
  connection: redis
});

worker.on('failed', async (job, error) => {
  if (job.attemptsMade >= 3) {
    await failedQueue.add('manual-review', {
      original_job_id: job.data.job_id,
      error: error.message,
      attempts: job.attemptsMade,
      failed_at: new Date().toISOString()
    });
  }
});
```

### 5. Timeout Configuration

**Application Layer:**
```typescript
const TIMEOUTS = {
  INITIATE_REQUEST: 5000,      // 5s - should be instant
  STATUS_CHECK: 30000,          // 30s - support long polling
  FILE_PROCESSING: 300000,      // 5min - max per file
  JOB_TOTAL: 7200000           // 2 hours - max job duration
};
```

**Redis Connection:**
```typescript
const redis = new Redis({
  host: 'localhost',
  port: 6379,
  connectTimeout: 10000,
  commandTimeout: 5000
});
```

## Proxy Configuration

### Cloudflare Configuration

**Option 1: Cloudflare Dashboard**
- Go to Speed → Optimization
- Disable "Auto Minify" for API routes
- Set "Browser Cache TTL" to "Respect Existing Headers"

**Option 2: Page Rules**
```
URL: api.yourdomain.com/v1/download/*
Settings:
- Browser Cache TTL: 1 hour
- Cache Level: Bypass
```

**Option 3: Cloudflare Workers (Advanced)**
```javascript
// cloudflare-worker.js
export default {
  async fetch(request) {
    const url = new URL(request.url);
    
    // Pass through status checks with extended timeout
    if (url.pathname.includes('/status/')) {
      return fetch(request, {
        cf: {
          timeout: 100000,
          cacheTtl: 0
        }
      });
    }
    
    // Fast response for initiate
    return fetch(request);
  }
};
```

### nginx Configuration
```nginx
upstream api_backend {
    server api:3000;
    keepalive 32;
}

server {
    listen 80;
    server_name api.yourdomain.com;
    
    # Initiate endpoint - fast response expected
    location /v1/download/initiate {
        proxy_pass http://api_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_read_timeout 10s;
        proxy_connect_timeout 5s;
        proxy_buffering off;
    }
    
    # Status endpoint - support polling
    location /v1/download/status {
        proxy_pass http://api_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_read_timeout 35s;
        proxy_buffering off;
        
        # Enable caching for status checks
        proxy_cache status_cache;
        proxy_cache_valid 200 2s;
        proxy_cache_key "$request_uri";
    }
    
    # Download endpoint - presigned URLs redirect
    location /v1/download/ {
        proxy_pass http://api_backend;
        proxy_http_version 1.1;
        proxy_read_timeout 5s;
        proxy_redirect off;
    }
    
    # General settings
    client_max_body_size 10M;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}

# Status cache configuration
proxy_cache_path /var/cache/nginx/status 
    levels=1:2 
    keys_zone=status_cache:10m 
    max_size=100m 
    inactive=60m;
```

## Frontend Integration (React/Next.js)

### Custom Hook: useDownload
```typescript
// hooks/useDownload.ts
import { useState, useEffect, useCallback } from 'react';

interface DownloadProgress {
  completed: number;
  total: number;
  percentage: number;
}

interface FileStatus {
  file_id: number;
  status: string;
  download_url?: string;
}

interface DownloadState {
  jobId: string | null;
  status: 'idle' | 'initiating' | 'queued' | 'processing' | 'completed' | 'failed' | 'cancelled';
  progress: DownloadProgress;
  files: FileStatus[];
  downloadUrl: string | null;
  error: string | null;
}

export function useDownload(fileIds: number[], pollInterval = 2000) {
  const [state, setState] = useState<DownloadState>({
    jobId: null,
    status: 'idle',
    progress: { completed: 0, total: 0, percentage: 0 },
    files: [],
    downloadUrl: null,
    error: null
  });

  const initiateDownload = useCallback(async () => {
    try {
      setState(prev => ({ ...prev, status: 'initiating' }));
      
      const response = await fetch('/v1/download/initiate', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ file_ids: fileIds })
      });
      
      if (!response.ok) throw new Error('Failed to initiate download');
      
      const data = await response.json();
      setState(prev => ({
        ...prev,
        jobId: data.job_id,
        status: 'queued',
        progress: { completed: 0, total: data.total_files, percentage: 0 }
      }));
    } catch (err) {
      setState(prev => ({
        ...prev,
        status: 'failed',
        error: err.message
      }));
    }
  }, [fileIds]);

  const cancelDownload = useCallback(async () => {
    if (!state.jobId) return;
    
    try {
      await fetch(`/v1/download/${state.jobId}`, { method: 'DELETE' });
      setState(prev => ({ ...prev, status: 'cancelled' }));
    } catch (err) {
      console.error('Failed to cancel:', err);
    }
  }, [state.jobId]);

  useEffect(() => {
    if (!state.jobId || state.status === 'completed' || state.status === 'failed' || state.status === 'cancelled') {
      return;
    }

    let isMounted = true;

    const pollStatus = async () => {
      try {
        const response = await fetch(`/v1/download/status/${state.jobId}`);
        if (!response.ok) throw new Error('Status check failed');
        
        const data = await response.json();
        
        if (!isMounted) return;
        
        setState(prev => ({
          ...prev,
          status: data.status,
          progress: data.progress,
          files: data.files,
          downloadUrl: data.download_url || null,
          error: data.error || null
        }));
      } catch (err) {
        if (!isMounted) return;
        setState(prev => ({
          ...prev,
          status: 'failed',
          error: err.message
        }));
      }
    };

    const intervalId = setInterval(pollStatus, pollInterval);
    pollStatus();

    return () => {
      isMounted = false;
      clearInterval(intervalId);
    };
  }, [state.jobId, state.status, pollInterval]);

  return {
    ...state,
    initiateDownload,
    cancelDownload,
    isLoading: ['initiating', 'queued', 'processing'].includes(state.status),
    isComplete: state.status === 'completed',
    isFailed: state.status === 'failed'
  };
}
```

### React Component
```typescript
// components/DownloadButton.tsx
import React from 'react';
import { useDownload } from '../hooks/useDownload';

interface Props {
  fileIds: number[];
}

export function DownloadButton({ fileIds }: Props) {
  const {
    status,
    progress,
    downloadUrl,
    error,
    initiateDownload,
    cancelDownload,
    isLoading,
    isComplete,
    isFailed,
    files
  } = useDownload(fileIds);

  if (isComplete && downloadUrl) {
    return (
      <div className="download-complete">
        <svg className="icon-success" viewBox="0 0 24 24">
          <path d="M9 16.17L4.83 12l-1.42 1.41L9 19 21 7l-1.41-1.41z"/>
        </svg>
        <a 
          href={downloadUrl} 
          className="download-link"
          download
        >
          Download Complete - Click to Save
        </a>
      </div>
    );
  }

  if (isLoading) {
    return (
      <div className="download-progress">
        <div className="progress-header">
          <span>Processing Files...</span>
          <button onClick={cancelDownload} className="btn-cancel">
            Cancel
          </button>
        </div>
        
        <div className="progress-bar">
          <div 
            className="progress-fill" 
            style={{ width: `${progress.percentage}%` }}
          />
        </div>
        
        <div className="progress-info">
          <span>{progress.completed} of {progress.total} files</span>
          <span>{progress.percentage}%</span>
        </div>
        
        <div className="file-list">
          {files.map(file => (
            <div key={file.file_id} className="file-item">
              <span className={`status-${file.status}`}>
                {file.status === 'completed' && '✓'}
                {file.status === 'processing' && '⟳'}
                {file.status === 'queued' && '○'}
              </span>
              <span>File {file.file_id}</span>
            </div>
          ))}
        </div>
      </div>
    );
  }

  if (isFailed) {
    return (
      <div className="download-error">
        <svg className="icon-error" viewBox="0 0 24 24">
          <path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 15h-2v-2h2v2zm0-4h-2V7h2v6z"/>
        </svg>
        <p className="error-message">{error}</p>
        <button onClick={initiateDownload} className="btn-retry">
          Try Again
        </button>
      </div>
    );
  }

  return (
    <button 
      onClick={initiateDownload} 
      className="btn-download"
    >
      Download {fileIds.length} Files
    </button>
  );
}
```

### CSS Styles
```css
/* components/DownloadButton.css */
.download-progress {
  padding: 20px;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  background: white;
}

.progress-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.progress-bar {
  height: 8px;
  background: #f0f0f0;
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 8px;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #4CAF50, #45a049);
  transition: width 0.3s ease;
}

.progress-info {
  display: flex;
  justify-content: space-between;
  font-size: 14px;
  color: #666;
  margin-bottom: 16px;
}

.file-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.file-item {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px;
  background: #f8f8f8;
  border-radius: 4px;
}

.status-completed {
  color: #4CAF50;
}

.status-processing {
  color: #2196F3;
  animation: spin 1s linear infinite;
}

.status-queued {
  color: #999;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

.download-complete,
.download-error {
  text-align: center;
  padding: 20px;
}

.icon-success {
  width: 48px;
  height: 48px;
  fill: #4CAF50;
  margin-bottom: 16px;
}

.icon-error {
  width: 48px;
  height: 48px;
  fill: #f44336;
  margin-bottom: 16px;
}

.download-link {
  display: inline-block;
  padding: 12px 24px;
  background: #4CAF50;
  color: white;
  text-decoration: none;
  border-radius: 4px;
  font-weight: 500;
}

.btn-cancel,
.btn-retry {
  padding: 8px 16px;
  border: 1px solid #ddd;
  background: white;
  border-radius: 4px;
  cursor: pointer;
}

.btn-download {
  padding: 12px 24px;
  background: #2196F3;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  font-weight: 500;
}

.btn-download:hover {
  background: #1976D2;
}
```

## Edge Cases & Considerations

### 1. User Closes Browser Mid-Download

**Solution**: Job continues processing server-side
```typescript
// User can resume by visiting: /downloads?jobId=abc123
// Or check their "Recent Downloads" page
```

### 2. Multiple Concurrent Downloads

**Solution**: Rate limit per user
```typescript
const userJobLimit = 3;

async function canUserCreateJob(userId: string): Promise<boolean> {
  const activeJobs = await redis.scard(`user:${userId}:jobs`);
  return activeJobs < userJobLimit;
}
```

### 3. Presigned URL Expiration

**Solution**: Generate new URL on demand
```typescript
app.get('/v1/download/refresh/:jobId', async (c) => {
  const jobId = c.req.param('jobId');
  const newUrl = await generatePresignedUrl(jobId);
  return c.json({ download_url: newUrl, expires_at: '...' });
});
```

## Cost Analysis

**Estimated AWS Costs (1000 jobs/day):**
- Redis (cache.t3.micro): $15/month
- EC2 Workers (t3.small x2): $30/month
- S3 Storage (100GB): $2.30/month
- Data Transfer (50GB): $4.50/month
**Total: ~$52/month**

**BullMQ vs AWS SQS:**
- BullMQ: Free (uses Redis)
- AWS SQS: $0.40 per million requests
- Winner: BullMQ (lower cost, more features)

## Monitoring & Alerts

**Key Metrics:**
```typescript
// Prometheus metrics
const metrics = {
  queue_depth: new Gauge({ name: 'download_queue_depth' }),
  job_duration: new Histogram({ name: 'download_job_duration_seconds' }),
  job_success_rate: new Counter({ name: 'download_job_success_total' }),
  job_failure_rate: new Counter({ name: 'download_job_failure_total' }),
  active_workers: new Gauge({ name: 'download_active_workers' })
};
```

**Alerts:**
- Queue depth > 100 for 5 minutes
- Job failure rate > 10% in 1 hour
- Average job duration > 5 minutes
- Worker process crashes

## Conclusion

The Polling Pattern with BullMQ provides an optimal balance of:

- **Reliability**: Jobs survive server restarts
- **Scalability**: Horizontal scaling of workers
- **Simplicity**: No WebSocket infrastructure
- **Cost**: Lower than managed queues
- **UX**: 2-3 second polling provides good feedback

This architecture handles 10-120 second downloads gracefully while working with any proxy configuration and providing excellent user experience through progress tracking and automatic retries.
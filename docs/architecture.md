# Architecture

zs3 is a single-file HTTP server implementing the S3 REST API.

## Overview

```
┌─────────────────────────────────────────────────┐
│                    main()                       │
│              (connection loop)                  │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│              handleConnection()                 │
│         (parse request, write response)         │
└─────────────────────┬───────────────────────────┘
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
┌──────────────────┐    ┌──────────────────┐
│   SigV4.verify() │    │     route()      │
│  (authenticate)  │    │(dispatch handler)│
└──────────────────┘    └────────┬─────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  handlePutObject│    │ handleGetObject │    │     ...         │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## Components

### S3Context

Holds configuration and provides path helpers.

```zig
const S3Context = struct {
    allocator: Allocator,
    data_dir: []const u8,
    access_key: []const u8,
    secret_key: []const u8,

    fn bucketPath(self, allocator, bucket) ![]const u8
    fn objectPath(self, allocator, bucket, key) ![]const u8
};
```

### Request/Response

Simple HTTP/1.1 parsing and serialization.

```zig
const Request = struct {
    method: []const u8,
    path: []const u8,
    query: []const u8,
    headers: StringHashMap,
    body: []const u8,
};

const Response = struct {
    status: u16,
    headers: ArrayList(Header),
    body: []const u8,
};
```

### SigV4

AWS Signature Version 4 implementation.

```
1. Parse Authorization header
2. Build canonical request (method, path, query, headers, payload hash)
3. Build string to sign (algorithm, timestamp, scope, canonical hash)
4. Derive signing key (HMAC chain: secret → date → region → service → request)
5. Calculate signature (HMAC of string to sign)
6. Compare with provided signature
```

### Handlers

Each S3 operation has a handler function that:
1. Validates input
2. Performs filesystem operations
3. Builds XML response

## Storage Layout

```
data/
├── bucket1/
│   ├── file.txt
│   └── folder/
│       └── nested.txt
├── bucket2/
│   └── ...
└── .uploads/
    └── {upload_id}/
        ├── 1
        ├── 2
        └── .meta
```

- Buckets are directories
- Objects are files
- Nested keys create nested directories
- Multipart uploads stored in `.uploads/` with metadata

## Memory Management

zs3 uses arena allocation per request:

```zig
var arena = std.heap.ArenaAllocator.init(allocator);
defer arena.deinit();
```

All allocations during request handling use the arena. When the request completes, everything is freed at once.

## Concurrency

zs3 is single-threaded. Connections are handled sequentially.

For concurrent access, the filesystem provides isolation - each request opens/closes files independently.

## Error Handling

Errors are caught at the handler level and converted to S3 XML error responses:

```zig
route(ctx, alloc, &req, &res) catch |err| {
    sendError(&res, 500, "InternalError", "Internal server error");
};
```

Individual handlers return specific errors (404, 400, etc.) via `sendError()`.

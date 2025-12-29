# zs3

A minimal S3-compatible object storage server in ~1100 lines of Zig. Zero dependencies.

## Why

| | zs3 | MinIO | RustFS |
|---|-----|-------|--------|
| Lines of code | ~1,100 | ~200,000 | ~80,000 |
| Dependencies | 0 | many | ~200 crates |
| Binary size | ~250KB | ~100MB | ~50MB |

Most S3 usage is PUT, GET, DELETE, LIST with basic auth. You don't need 200k lines of code for that.

## Features

- Full AWS SigV4 authentication (works with aws CLI, boto3, any S3 SDK)
- Core operations: PUT, GET, DELETE, HEAD, LIST (v2)
- Multipart upload for large files
- Range requests for streaming/seeking
- Recursive listing with delimiter and pagination
- Input validation (bucket names, key length, request size limits)
- Zero dependencies (only Zig standard library)
- Single file, easy to audit and modify

## Quick Start

```bash
zig build -Doptimize=ReleaseFast
./zig-out/bin/zs3
```

Server listens on port 9000, stores data in `./data`.

## Usage

```bash
export AWS_ACCESS_KEY_ID=minioadmin
export AWS_SECRET_ACCESS_KEY=minioadmin

# Create bucket
aws --endpoint-url http://localhost:9000 s3 mb s3://mybucket

# Upload
aws --endpoint-url http://localhost:9000 s3 cp file.txt s3://mybucket/

# List
aws --endpoint-url http://localhost:9000 s3 ls s3://mybucket/ --recursive

# Download
aws --endpoint-url http://localhost:9000 s3 cp s3://mybucket/file.txt ./

# Delete
aws --endpoint-url http://localhost:9000 s3 rm s3://mybucket/file.txt
```

Works with any S3 SDK:

```python
import boto3

s3 = boto3.client('s3',
    endpoint_url='http://localhost:9000',
    aws_access_key_id='minioadmin',
    aws_secret_access_key='minioadmin'
)

s3.create_bucket(Bucket='test')
s3.put_object(Bucket='test', Key='hello.txt', Body=b'world')
print(s3.get_object(Bucket='test', Key='hello.txt')['Body'].read())
```

## Supported Operations

| Operation | Supported |
|-----------|-----------|
| PutObject | yes |
| GetObject | yes |
| DeleteObject | yes |
| HeadObject | yes |
| ListObjectsV2 | yes |
| ListBuckets | yes |
| CreateBucket | yes |
| DeleteBucket | yes |
| Multipart Upload | yes |
| Range Requests | yes |
| SigV4 Auth | yes |

## Not Supported

- Versioning
- Lifecycle policies
- Bucket policies / ACLs
- Cross-region replication
- Server-side encryption (use disk encryption instead)
- Pre-signed URLs
- Bucket notifications
- Object tagging
- Object lock

If you need these, use MinIO or a cloud provider.

## Configuration

Edit `main.zig`:

```zig
const ctx = S3Context{
    .allocator = allocator,
    .data_dir = "data",
    .access_key = "minioadmin",
    .secret_key = "minioadmin",
};

const address = net.Address.parseIp4("0.0.0.0", 9000)
```

## Limits

| Limit | Value |
|-------|-------|
| Max header size | 8 KB |
| Max body size | 5 GB |
| Max key length | 1024 bytes |
| Bucket name length | 3-63 characters |

## Building

Requires Zig 0.15 or later.

```bash
# Debug build
zig build

# Release build (~250KB binary)
zig build -Doptimize=ReleaseFast

# Cross-compile
zig build -Dtarget=x86_64-linux-musl -Doptimize=ReleaseFast
zig build -Dtarget=aarch64-linux-musl -Doptimize=ReleaseFast
```

## Architecture

```
main.zig
  main()                      server loop
  S3Context                   config and path helpers
  Request/Response            HTTP parsing and writing
  route()                     method + path dispatch
  SigV4                       AWS signature verification
    parseAuthHeader()
    buildCanonicalRequest()
    buildStringToSign()
    calculateSignature()
  Handlers
    handlePutObject()
    handleGetObject()
    handleDeleteObject()
    handleHeadObject()
    handleListObjects()
    handleListBuckets()
    handleCreateBucket()
    handleDeleteBucket()
    handleInitiateMultipart()
    handleUploadPart()
    handleCompleteMultipart()
    handleAbortMultipart()
  Helpers
    isValidBucketName()
    isValidKey()
    uriEncode()
    sortQueryString()
    parseRange()
    xmlEscape()
```

Storage layout:

```
data/
  mybucket/
    file.txt
    folder/
      nested.txt
  .uploads/
    {upload_id}/
      1
      2
      .meta
```

## Security

- Full SigV4 signature verification
- Input validation on bucket names and object keys
- Request size limits
- No shell commands, no eval, no external network calls
- Single file, easy to audit

TLS is not included. Use a reverse proxy (nginx, caddy) for HTTPS.

## Use Cases

- Local development (replace localstack/minio)
- CI/CD artifact storage
- Self-hosted backups
- Embedded storage for appliances
- Learning how S3 protocol works

## License

[WTFPL](LICENSE)

## Contributing

Read it, modify it, make it yours.

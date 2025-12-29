# Configuration

zs3 is configured by editing `main.zig` and rebuilding.

## Server Settings

```zig
const address = net.Address.parseIp4("0.0.0.0", 9000)
```

- `0.0.0.0` - Listen on all interfaces (use `127.0.0.1` for localhost only)
- `9000` - Port number

## Authentication

```zig
const ctx = S3Context{
    .allocator = allocator,
    .data_dir = "data",
    .access_key = "minioadmin",
    .secret_key = "minioadmin",
};
```

- `data_dir` - Directory for storing buckets and objects
- `access_key` - AWS access key ID
- `secret_key` - AWS secret access key

## Limits

Edit constants at top of `main.zig`:

```zig
const MAX_HEADER_SIZE = 8 * 1024;          // 8 KB
const MAX_BODY_SIZE = 5 * 1024 * 1024 * 1024;  // 5 GB
const MAX_KEY_LENGTH = 1024;               // bytes
const MAX_BUCKET_LENGTH = 63;              // characters
```

## TLS/HTTPS

zs3 does not include TLS. Use a reverse proxy:

### nginx

```nginx
server {
    listen 443 ssl;
    server_name s3.example.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://127.0.0.1:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Caddy

```
s3.example.com {
    reverse_proxy localhost:9000
}
```

## Environment Variables

zs3 does not read environment variables. All configuration is compile-time.

For dynamic configuration, modify `main.zig` to read from environment:

```zig
const access_key = std.posix.getenv("ZS3_ACCESS_KEY") orelse "minioadmin";
const secret_key = std.posix.getenv("ZS3_SECRET_KEY") orelse "minioadmin";
```

## Multiple Users

zs3 supports only one set of credentials. For multiple users, run multiple instances or add user lookup to `SigV4.verify()`.

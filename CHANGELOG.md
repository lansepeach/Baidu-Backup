# Changelog

## v0.1.0 (Initial release)

Highlights:
- Streaming, zero-intermediate-file backup pipeline using tar | split
- Parallel uploads via xargs to saturate bandwidth
- Robust Python uploader with:
  - OAuth token management with automatic refresh (oob flow)
  - Double MD5 validation of chunks before upload
  - Chunked uploads and server-side merge
  - Automatic rotation of old backup sets via filemanager API
- Clear logging designed for unattended jobs
- Example automation via cron and systemd
- Operational documentation and example configuration

Breaking changes:
- None (first public version)

Usage notes:
- Place Baidu Open Platform credentials in environment variables BAIDU_APP_KEY and BAIDU_SECRET_KEY
- Adjust performance knobs (SPLIT_SIZE, PARALLEL_UPLOADS) to match your host
- Remote backup path for Baidu must start with /apps/<your-app-name>

Known limitations:
- Only Baidu Netdisk backend is supported in this release
- 123pan backend is planned/experimental; see README for credential preparation steps


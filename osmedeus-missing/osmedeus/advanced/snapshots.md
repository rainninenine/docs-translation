> ## Documentation Index
> Fetch the complete documentation index at: https://docs.osmedeus.org/llms.txt
> Use this file to discover all available pages before exploring further.

# Snapshots

Export and import workspaces as compressed ZIP archives.

## Overview

Snapshots enable:

* Backup and restore workspaces
* Share scan results
* Migrate between machines
* Archive completed assessments

## Export

### CLI

```bash theme={null}
# Export workspace
osmedeus snapshot export example.com

# Custom output path
osmedeus snapshot export example.com -o /backups/example.zip

# Export to specific directory
osmedeus snapshot export example.com -o ~/archives/
```

### API

```bash theme={null}
curl -X POST http://localhost:8002/osm/api/snapshots/export \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"workspace": "example.com"}'
```

### Contents

Exported ZIP includes:

```
example.com-20240115-143022.zip
├── workspace/                    # All workspace files
│   ├── subdomain-enum/
│   ├── http-probe/
│   └── ...
├── metadata.json                 # Export metadata
└── database.json                 # Database records
    ├── assets
    ├── vulnerabilities
    └── runs
```

## Import

### CLI

```bash theme={null}
# Import from local file
osmedeus snapshot import ~/backup.zip

# Import from URL
osmedeus snapshot import https://example.com/workspace.zip

# Force overwrite existing
osmedeus snapshot import ~/backup.zip --force

# Files only (skip database)
osmedeus snapshot import ~/backup.zip --skip-db
```

### API

```bash theme={null}
# Import from URL
curl -X POST http://localhost:8002/osm/api/snapshots/import \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"source": "https://example.com/backup.zip"}'

# Import with options
curl -X POST http://localhost:8002/osm/api/snapshots/import \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "source": "/path/to/backup.zip",
    "force": true,
    "skip_db": false
  }'
```

## List Snapshots

### CLI

```bash theme={null}
osmedeus snapshot list
```

### API

```bash theme={null}
curl http://localhost:8002/osm/api/snapshots \
  -H "Authorization: Bearer $TOKEN"
```

### Output

```
Available Snapshots:

┌────────────────────────────────────────────┬────────────┬─────────────────────┐
│ Filename                                   │ Size       │ Created             │
├────────────────────────────────────────────┼────────────┼─────────────────────┤
│ example.com-20240115-143022.zip            │ 12.5 MB    │ 2024-01-15 14:30:22 │
│ target.org-20240114-091500.zip             │ 8.2 MB     │ 2024-01-14 09:15:00 │
└────────────────────────────────────────────┴────────────┴─────────────────────┘
```

## Download

```bash theme={null}
# Via API
curl http://localhost:8002/osm/api/snapshots/download/example.com-backup.zip \
  -H "Authorization: Bearer $TOKEN" \
  -o local-backup.zip
```

## Delete

```bash theme={null}
curl -X DELETE http://localhost:8002/osm/api/snapshots/example.com-backup.zip \
  -H "Authorization: Bearer $TOKEN"
```

## Storage Location

Default: `~/osmedeus-base/snapshots/`

Configure in `osm-settings.yaml`:

```yaml theme={null}
environments:
  storages: "{{base_folder}}/snapshots"
```

## Use Cases

### Backup Before Destructive Scan

```bash theme={null}
# Backup current state
osmedeus snapshot export example.com

# Run potentially destructive scan
osmedeus run -f aggressive-scan -t example.com

# Restore if needed
osmedeus snapshot import ~/osmedeus-base/snapshots/example.com-*.zip --force
```

### Share Results

```bash theme={null}
# Export workspace
osmedeus snapshot export client-assessment -o /tmp/results.zip

# Share via cloud storage
aws s3 cp /tmp/results.zip s3://bucket/results.zip
```

### Migrate to Another Machine

```bash theme={null}
# On source machine
osmedeus snapshot export example.com -o /tmp/workspace.zip
scp /tmp/workspace.zip user@target:/tmp/

# On target machine
osmedeus snapshot import /tmp/workspace.zip
```

### Archive Completed Projects

```bash theme={null}
# Export and compress
osmedeus snapshot export project-2024 -o /archives/project-2024.zip

# Remove workspace
rm -rf ~/osmedeus-base/workspaces/project-2024

# Restore later if needed
osmedeus snapshot import /archives/project-2024.zip
```

## Metadata

Each snapshot includes `metadata.json`:

```json theme={null}
{
  "version": "1.0",
  "workspace": "example.com",
  "created_at": "2024-01-15T14:30:22Z",
  "osmedeus_version": "2.0.0",
  "files_count": 156,
  "total_size": 13107200,
  "records": {
    "assets": 423,
    "vulnerabilities": 15,
    "runs": 3
  }
}
```

## Database Records

The `database.json` contains:

```json theme={null}
{
  "assets": [
    {
      "id": 1,
      "workspace": "example.com",
      "asset_value": "sub.example.com",
      "url": "https://sub.example.com",
      "status_code": 200,
      ...
    }
  ],
  "vulnerabilities": [
    {
      "id": 1,
      "workspace": "example.com",
      "title": "SQL Injection",
      "severity": "high",
      ...
    }
  ],
  "runs": [
    {
      "id": "run-abc123",
      "workflow_name": "full-recon",
      "target": "example.com",
      ...
    }
  ]
}
```

## Import Options

### Force Overwrite

```bash theme={null}
osmedeus snapshot import backup.zip --force
```

* Deletes existing workspace
* Removes existing database records
* Imports fresh from snapshot

### Skip Database

```bash theme={null}
osmedeus snapshot import backup.zip --skip-db
```

* Only imports files
* Preserves existing database records
* Useful for file-only updates

## Cloud Storage

### Export to S3

```bash theme={null}
# Export locally
osmedeus snapshot export example.com -o /tmp/backup.zip

# Upload to S3
aws s3 cp /tmp/backup.zip s3://bucket/snapshots/

# Or configure storage in settings
storage:
  enabled: true
  provider: s3
  bucket: osmedeus-snapshots
```

### Import from S3

```bash theme={null}
# Generate presigned URL
URL=$(aws s3 presign s3://bucket/snapshots/backup.zip)

# Import from URL
osmedeus snapshot import "$URL"
```

## Best Practices

1. **Regular backups** - Schedule snapshot exports
2. **Verify after import** - Check file integrity
3. **Use descriptive names** - Include date and purpose
4. **Secure storage** - Encrypt sensitive snapshots
5. **Clean old snapshots** - Manage storage usage

## Troubleshooting

### Import fails

```bash theme={null}
# Check file integrity
unzip -t backup.zip

# Try with verbose output
osmedeus snapshot import backup.zip -v
```

### Missing files after import

```bash theme={null}
# Check if skip-db was used
# Re-import with database
osmedeus snapshot import backup.zip
```

### Large snapshot size

```bash theme={null}
# Check workspace contents
du -sh ~/osmedeus-base/workspaces/example.com/*

# Clean unnecessary files before export
rm -rf ~/osmedeus-base/workspaces/example.com/temp/
```

## Next Steps

* [Snapshot CLI](../cli/snapshot.md) - CLI reference
* [API Overview](../api/overview.md) - Snapshot endpoints
* [Workspaces](../concepts/workflows.md) - Workspace structure

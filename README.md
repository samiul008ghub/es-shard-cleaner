# es-shard-cleaner

Safely delete all indices with unassigned shards in a secure Elasticsearch environment using HTTPS and Basic Auth, without requiring SSL certificates.

---

## Problem Statement

Elasticsearch clusters can experience **red health status** due to many unassigned shards, causing search failures and degraded performance. Errors similar to the following may be observed:

```
org.elasticsearch.action.search.SearchPhaseExecutionException:
Search rejected due to missing shards [[metrics-endpoint.metadata_current_default][0]]
```

Cluster statistics may show:
```json
{ 
  "status": "red", 
  "unassigned_shards": 696, 
  "active_shards_percent_as_number": 30.0 
}
```

Unassigned shards may result from corrupted indices, node failures, or misconfiguration.

---

## Objective

Automatically identify and delete all Elasticsearch indices with unassigned shards, helping restore cluster health—without requiring SSL certificates.

---

## How It Works

- Queries Elasticsearch for indices with unassigned shards.
- Lists those indices for your review and confirmation.
- Permanently deletes the listed indices upon confirmation.

---

## Prerequisites

- Elasticsearch accessible via HTTPS.
- Credentials with permissions to delete indices.
- Bash shell with `curl` and `awk` installed.
- Awareness that **deleted indices cannot be recovered**.

---

## Usage

### 1. Save the script as `delete_unassigned_indices.sh`

```bash
#!/bin/bash

# === CONFIGURATION ===
ES_HOST="https://your-elasticsearch-host:9200"     # Your Elasticsearch URL
ES_USER="your_username"                            # Username with delete access
ES_PASS="your_password"                            # Password
# ======================

# Test connection
if ! curl -s -k -u "$ES_USER:$ES_PASS" "$ES_HOST" > /dev/null; then
  echo "[x] Could not connect to Elasticsearch at $ES_HOST"
  exit 1
fi

# Find indices with unassigned shards
echo "[*] Looking for unassigned shards..."
indices=$(curl -s -k -u "$ES_USER:$ES_PASS" "$ES_HOST/_cat/shards?h=index,state" \
  | awk '$2 == "UNASSIGNED" {print $1}' \
  | sort -u)

if [[ -z "$indices" ]]; then
  echo "[✓] No unassigned shards found. Cluster is clean."
  exit 0
fi

echo "[!] These indices have unassigned shards and may be corrupted:"
echo "$indices" | sed 's/^/ - /'

read -p "Type 'yes' to delete them permanently: " confirm
if [[ "$confirm" != "yes" ]]; then
  echo "[x] Aborting deletion."
  exit 1
fi

for index in $indices; do
  echo "[*] Deleting: $index"
  response=$(curl -s -k -X DELETE -u "$ES_USER:$ES_PASS" "$ES_HOST/$index")
  if echo "$response" | grep -q '"acknowledged":true'; then
    echo "[✓] Deleted $index"
  else
    echo "[!] Failed to delete $index — response: $response"
  fi
done

echo "[✓] Done. Check cluster health with:"
echo "curl -s -k -u \$ES_USER:\$ES_PASS \$ES_HOST/_cluster/health?pretty"
```

---

### 2. Make the script executable

```bash
chmod +x delete_unassigned_indices.sh
```

---

### 3. Run the script

```bash
./delete_unassigned_indices.sh
```

---

### 4. Confirm deletion by typing `yes` when prompted.

---

### 5. Check cluster health

```bash
curl -s -k -u your_username:your_password https://your-elasticsearch-host:9200/_cluster/health?pretty
```

---

## Important Notes

- The script **permanently deletes** indices with unassigned shards.
- Always back up important data before running.
- Exercise caution in production environments.
- Consider Elasticsearch’s [Index Lifecycle Management](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html) for automated index handling.

---

# Demo Walkthrough

Complete the [Environment Setup](README.md) before starting.

## Act 1: Redis Baseline

### 1.1 Verify redis-client-app points to Redis

```bash
curl -sk "https://redis-client-app-rhdg-poc.apps.<your-cluster-domain>/api/info" | jq .
```

Expected output:

```json
{
  "backend_host": "redis",
  "backend_port": 6379
}
```

### 1.2 Open the redis-client-app web UI

Open in a browser:

```
https://redis-client-app-rhdg-poc.apps.<your-cluster-domain>
```

The top bar shows connection status (green dot = connected) and six cards: Cluster, Cache, Mode, Host, Port, Status.

### 1.3 String operations (Strings tab)

The UI opens on the **Strings** tab by default.

1. Type `demo:greeting` in the **Key** field and `hello from redis` in the **Value** field. Click **SET**.
2. Type `demo:patient` in Key, `John Smith` in Value. Click **SET**.
3. Type `demo:room` in Key, `ICU-42` in Value. Click **SET**.

After each SET, observe:

- The entry appears immediately in the **Cache Entries** table below
- The **Event Log** at the bottom shows the operation in real time (Pub/Sub via SSE)
- The entry count updates automatically

4. Type `demo:greeting` in Key and click **GET**. The result line shows `GET demo:greeting -> "hello from redis"` with the remaining TTL.
5. Click **DEL** on any entry in the table. The entry disappears and a red event appears in the log.

### 1.4 Hash operations (Hashes tab)

Click the **Hashes** tab.

1. Type `demo:user` in the **Key** field.
2. In the first field row, type `name` and `John` as the field name and value.
3. Click **+ Field** to add a second row. Type `role` and `admin`.
4. Click **HSET**. The result shows `HSET demo:user -> 2 fields`. The entry appears in the table with type `hash`.
5. Check the **Event Log** - it shows `HSET demo:user` in real time (Pub/Sub via SSE).
6. Click **HGETALL**. The result shows all fields: `{name="John", role="admin"}`.

### 1.5 Counter operations (Counters tab)

Click the **Counters** tab.

1. Type `demo:visits` in the **Key** field.
2. Click **INCR** three times. Each click shows the incremented value: 1, 2, 3.
3. Click **DECR** once. The value goes back to 2.
4. Check the **Event Log** - each INCR/DECR generates a Pub/Sub event showing the updated value.

### 1.6 List all keys (Keys tab)

Click the **Keys** tab.

1. Leave the pattern as `*` and click **KEYS**. All entries created so far are listed with a count.
2. Change the pattern to `demo:*` and click **KEYS** again. Same result (all keys use the `demo:` prefix).

Every write operation (SET, HSET, INCR, DECR, DEL) generates a Pub/Sub event visible in the Event Log. Read operations (GET, HGETALL, KEYS) do not.

This establishes the baseline: a standard Spring Boot app using Jedis to talk to Redis. All operations (SET, GET, HSET, HGETALL, INCR, DECR, KEYS) work through the web UI.

---

## Act 2: Start external-cache-client

### 2.1 Start the Quarkus app in a separate terminal

```bash
cd apps/external-cache-client
DG_PASSWORD=<your-healthcare-password> mvn quarkus:dev
```

Expected output in the terminal:

```
Listening for transport dt_socket at address: 5005
__  ____  __  _____   ___  __ ____  ______
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/
...
Installed features: [cdi, rest, rest-jackson, smallrye-health]
```

### 2.2 Verify the connection

```bash
curl -s http://localhost:8080/api/cache/info | jq .
```

Expected output:

```json
{
  "host": "respcache.apps.<your-cluster-domain>",
  "port": 443,
  "connected": true
}
```

The terminal will also show Pub/Sub subscription messages as events arrive.

---

## Act 3: Live Switch to Data Grid

### 3.1 Patch redis-client-app to point to Data Grid

```bash
oc patch deployment redis-client-app -n rhdg-poc --patch-file patches/patch-redis-client-to-datagrid.yaml
oc rollout status deployment/redis-client-app -n rhdg-poc
```

Wait for the rollout to complete (~15-20 seconds).

### 3.2 Verify the switch

```bash
curl -sk "https://redis-client-app-rhdg-poc.apps.<your-cluster-domain>/api/info" | jq .
```

Expected output:

```json
{
  "backend_host": "healthcare",
  "backend_port": 11222
}
```

### 3.3 String operations (Strings tab)

Refresh the browser. The connection cards now show the Data Grid endpoint (host: `healthcare`, port: `11222`).

Keep the **external-cache-client terminal** visible alongside the browser. From this point on, every write operation generates Pub/Sub events visible in both places simultaneously.

1. Type `demo:greeting` in the **Key** field and `hello from datagrid` in the **Value** field. Click **SET**.
2. Check the **Event Log** in the web UI - it shows `SET demo:greeting` in real time.
3. Check the **external-cache-client terminal** - the same event appears there. Two different frameworks (Spring Boot + Quarkus), same Pub/Sub channel, same Data Grid cluster.
4. Type `demo:greeting` in Key and click **GET**. The result shows `GET demo:greeting -> "hello from datagrid"` with the remaining TTL.

### 3.4 Hash operations (Hashes tab)

Click the **Hashes** tab.

1. Type `demo:user` in the **Key** field.
2. In the first field row, type `name` and `Jane` as the field name and value.
3. Click **+ Field** to add a second row. Type `role` and `doctor`.
4. Click **HSET**. The result shows `HSET demo:user -> 2 fields`.
5. Check both the **Event Log** and the **external-cache-client terminal** - both show the `HSET demo:user` event.
6. Click **HGETALL**. The result shows all fields: `{name="Jane", role="doctor"}`.

### 3.5 Counter operations (Counters tab)

Click the **Counters** tab.

1. Type `demo:visits` in the **Key** field.
2. Click **INCR** three times. The counter starts from 1 (this is a new counter in Data Grid, independent from the one created in Redis).
3. Each INCR generates a Pub/Sub event visible in both the **Event Log** and the **external-cache-client terminal**.

### 3.6 List all keys (Keys tab)

Click the **Keys** tab.

1. Leave the pattern as `*` and click **KEYS**. All entries created in Data Grid are listed.
2. DEL one entry from the table. The deletion event appears in both the **Event Log** and the **external-cache-client terminal**.

All operations work identically to Act 1. The app code did not change - only the backend endpoint was patched from Redis to Data Grid.

---

## Act 4: Data Grid Console

### 4.1 Open the Data Grid Console

Open in a browser:

```
https://healthcare-external-rhdg-poc.apps.<your-cluster-domain>/console/
```

Log in with the `ops-admin` credentials defined in `manifests/03-healthcare-credentials.yaml`.

### 4.2 Inspect the cluster

Navigate to:

1. **Data Container** - verify both nodes are online and healthy
2. **respCache** - open the cache and inspect the entries created from the web UI (`demo:greeting`, `demo:user`, `demo:visits`)
3. **Cache metrics** - observe hit/miss ratios, entry count, memory usage

The operations team can monitor cache contents, cluster health, and metrics without needing access to the application pods.

---

## Act 5: .NET + Gatling (optional)

### 5.1 Start dotnet-hash-test in a separate terminal

```bash
cd apps/dotnet-hash-test
export DG_HOST=respcache.apps.<your-cluster-domain>
export DG_PORT=443
export DG_USER=app-healthcare
export DG_PASSWORD=<your-healthcare-password>
dotnet run --launch-profile http
```

Expected output:

```
Now listening on: http://localhost:5100
Application started. Press Ctrl+C to shut down.
```

### 5.2 Verify the connection

```bash
curl -s http://localhost:5100/health | jq .
```

Expected output:

```json
{
  "status": "connected",
  "host": "respcache.apps.<your-cluster-domain>"
}
```

### 5.3 Test a cache lookup

```bash
curl -s http://localhost:5100/catalog/table_01/field_01 | jq .
```

Expected output shows the cached value with `source: "cache"`. If the entry expired or was not seeded, it falls back to SQLite (`source: "database"`).

### 5.4 Run the Gatling load test

In a separate terminal:

```bash
cd apps/gatling-load-test
mvn gatling:test
```

The test runs three scenarios:

1. **Single HGET** - ramp 1 to 10 req/s over 30s, hold for 30s
2. **Multi HGET** (20 parallel) - ramp 1 to 5 req/s over 30s, hold for 60s
3. **Mixed workload** - 80% single / 20% multi, ramp to 50 req/s

Expected result: p95 response time < 5s, failed requests < 5%.

The HTML report is generated in `apps/gatling-load-test/target/gatling/`. Open the `index.html` file in a browser to review throughput, latency percentiles, and error rates.

---

## Reset

Switch redis-client-app back to Redis:

```bash
oc patch deployment redis-client-app -n rhdg-poc --patch-file patches/patch-redis-client-to-redis.yaml
oc rollout status deployment/redis-client-app -n rhdg-poc
```

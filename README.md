# apws-logstash

A minimal Logstash setup that ingests weather sensor readings over HTTP and
forwards them to OpenSearch, routed into per-sensor indices.

## Architecture

```
sensor(s) --HTTP POST(JSON)--> Logstash:5044 --> OpenSearch
```

- **Input**: `http` plugin listens on `0.0.0.0:5044` and expects JSON bodies
  (ECS compatibility disabled).
- **Filter**: a `ruby` filter copies the ingest-time `@timestamp` into a
  separate `timestamp` field.
- **Routing**: events are routed by the `[name]` field:
  - `"near_the_plant"` → index `weather-YYYY.MM.dd`
  - anything else (including `"hygrometer"`) → index `hygrometer-YYYY.MM.dd`
- **Output**: `logstash-output-opensearch` plugin writes to
  `https://opensearch:9200`.

## Prerequisites

- Docker
- An OpenSearch instance reachable at hostname `opensearch` on port `9200`
  (e.g. on the same Docker network)
- An `arm64` host, since the image is currently pinned to
  `arm64v8/logstash:8.19.18`

## Build

```bash
cd src
docker build -t logstash .
```

## Run

The pipeline requires OpenSearch credentials to be supplied as environment
variables:

```bash
docker run -d --name logstash --rm \
  --network opensearch-net \
  -p 5044:5044 \
  -e OPENSEARCH_USER=admin \
  -e OPENSEARCH_PASSWORD=<your-password> \
  logstash
```

Adjust `--network` to whichever Docker network your OpenSearch container is
attached to, so that the `opensearch` hostname resolves.

## Sending data

Send a JSON payload to the HTTP input. The `name` field determines the
target index — only `"near_the_plant"` routes to the `weather-*` index,
everything else (including no `name` at all) ends up in `hygrometer-*`:

```bash
curl -X POST http://localhost:5044 \
  -H "Content-Type: application/json" \
  -d '{"name": "near_the_plant", "temperature": 21.5, "humidity": 47}'
```

## Configuration reference

| Variable               | Required | Description                          |
|-------------------------|----------|---------------------------------------|
| `OPENSEARCH_USER`       | yes      | OpenSearch username                   |
| `OPENSEARCH_PASSWORD`   | yes      | OpenSearch password                   |
| `XPACK_MONITORING_ENABLED` | no (set to `false` in image) | Disables X-Pack monitoring |


## License

MIT — see [LICENSE](LICENSE).

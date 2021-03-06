# Nodestack Demo

## Usage

```sh
./init-pki.md # generate TLS certificates
docker-compose up # start stack
```

## Basic Connection Test

Connection to LB (Nginx):

```sh
# self signed certs, either use `--insecure` or pass CA cert
curl --insecure https://localhost:4430
curl --cacert tls/ca/root.crt https://localhost:4430
```

TLS client cert validation:

```sh
# will result in SSL handshake failure (client cert is missing)
curl --cacert tls/ca/root.crt https://localhost:3000
```

## Example App

### Counter Example (Redis)

```sh
curl --cacert tls/ca/root.crt https://localhost:4430/counter
```

### Shout Example (Postgres)

```sh
# create (repeat as needed)
curl \
  --cacert tls/ca/root.crt \
  -XPOST \
  -H'Content-type: application/json' \
  -d'{"text": "Hello world!"}' \
  https://localhost:4430/shout

# list
curl \
  --cacert tls/ca/root.crt \
  https://localhost:4430/shout
```

## Verify SSL Certs

```sh
# nginx 
openssl s_client -connect localhost:4430 -showcerts 2>/dev/null </dev/null | openssl x509 -noout -text
# node app
openssl s_client -connect localhost:3000 -showcerts 2>/dev/null </dev/null | openssl x509 -noout -text
```

## Considerations

### TLS

- Normally we would need a TLS cert signed by a public authority on the public facing ingress points
- E.g. LetsEncrypt or AWS's ACM would be good options for free, automated certificate provisioning
- Internally we can use self-signed certs
- In any case, keys need to be stored in a secure location
- Use intermediate certs to limit exposure of CA keys
- Storage: e.g. Hashicorp Vault, AWS Secret Secrets Manager

### Redis & TLS

- Redis doesn't support TLS by itself
- For options, see https://redis.io/topics/encryption
- Docker: multi-process container
- Kubernetes: pod with sidecar

## Resource Planning

### CPU

- Node can't use more than one core, scale by running more instances
- Nginx as proxy is relatively lighweight on CPU, can be neglected
- Postgres: high CPU usage for indexing, search, aggregation (esp. with high number of connections)
- Redis: k/v store, not many compute heavy operations

### Memory

- Node: depends on workload, generally tends to be relatively lightweight
- Postgres is relatively light on mem, likely to run out of CPU first
- Redis = in memory DB, might need a lot of memory based on workload

### Scaling

- Assuming app is stateless, just run more instances
- Redis can be clustered, uses sharding to split workload

### Bottleneck: Postgres Database

- Single DB instance to power a fleet of backends
- Can't be sharded trivially, can't scale horizontally
- Large number of connections -> CPU pressure

### Alternatives/Mitigation

Option A: Use NoSQL DB

- E.g. MongoDB, DynamoDB
- Pros: can be sharded, HA, horizontally scalable, high partition tolerance, 
- Cons: different programming model, not suitable for all loads, eventually consistent, limited support for transactions

Option B: Scale Postgres

- Replication, offload reads to replicas
- Delegate write transactions into workers
- Writes won't be immediately visible on replicas
- Effectively becomes eventually consistent
- Partition tolerant only for reads (queues mitigate the problem)
- Consider external connection pooling (https://pgbouncer.github.io/)
- Offload read-heavy workloads (search, reporting) into specialist apps (Elastic, BI systems)

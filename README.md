# LMS RabbitMQ Observability Stack

RabbitMQ + Prometheus + Grafana + node_exporter. Config-only. One command start.

## Quick Start

```bash
# 1. Copy env template and fill secrets
cp .env.example .env
$EDITOR .env

# 2. Start stack
docker compose up -d

# 3. Verify all healthy
docker compose ps
```

Services come up in ~30s. RabbitMQ healthcheck gates Grafana start.

## Ports

| Service | URL |
|---------|-----|
| RabbitMQ Management | http://localhost:15672 |
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:3000 |

## Env Setup

Copy `.env.example` to `.env` and set at minimum:

```
RABBITMQ_DEFAULT_USER=lms_user
RABBITMQ_DEFAULT_PASS=<strong password>
RABBITMQ_DEFAULT_VHOST=lms
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=<strong password>
```

Memory watermark (`RABBITMQ_VM_MEMORY_HIGH_WATERMARK=0.6`) and disk limit (`RABBITMQ_DISK_FREE_LIMIT=2GB`) have safe defaults. Tune for your host.

## Verify Stack Healthy

```bash
# All services running
docker compose ps

# RabbitMQ plugins loaded (should list rabbitmq_prometheus)
docker compose exec rabbitmq rabbitmq-plugins list --enabled

# Prometheus targets all UP
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep '"health"'

# Grafana reachable
curl -s http://localhost:3000/api/health
```

Expected healthy signals:
- `docker compose ps` → all `healthy`
- Rabbit Management UI shows plugin `Prometheus` listed
- Prometheus `/targets` → 3 targets all `"health":"up"` (prometheus, rabbitmq, node_exporter)
- Grafana dashboard `LMS RabbitMQ Observability` auto-loaded; `queue` variable shows `lms.*` queues

## Grafana Dashboard

Auto-provisioned on first boot. Navigate to **Dashboards → LMS RabbitMQ Observability**.

Panels:
- **Ready Messages** — stat, thresholded green/yellow/red
- **Unacknowledged Messages** — stat, thresholded
- **Consumers** — stat, red if zero
- **Rabbit Memory Used** — stat MB
- **Queue Depth Over Time** — timeseries ready + unacked per queue
- **Publish / Deliver Rates** — timeseries msg/s
- **Host CPU Usage** — timeseries %
- **Host Memory** — timeseries used/available bytes

`Queue` variable filters to `lms.*` queues by default. Select `All` to see everything.

## Fair-Dispatch Policy

### Problem

Without prefetch limits Rabbit pushes all queued messages to the first available consumer. Slow worker gets flooded; fast worker sits idle.

### Solution: `prefetch_count=1` + manual ack

Configure every LMS worker with:

```python
# pika example
channel.basic_qos(prefetch_count=1)
channel.basic_consume(queue="lms.tasks", on_message_callback=handle, auto_ack=False)

def handle(ch, method, properties, body):
    process(body)
    ch.basic_ack(delivery_tag=method.delivery_tag)  # ack AFTER work done
```

```java
// spring-amqp: set prefetchCount=1 on the listener container factory
@RabbitListener(queues = "lms.tasks")
public void handle(Message msg, Channel channel) throws IOException {
    process(msg);
    channel.basicAck(msg.getMessageProperties().getDeliveryTag(), false);
}
```

### How It Works

1. `prefetch_count=1` — Rabbit holds next message until worker acks current one.
2. Fast worker acks sooner → receives more messages naturally.
3. Slow worker acks slowly → Rabbit routes remaining load to faster workers.
4. Result: adaptive load distribution, no central scheduler needed.

### Worker Connection Parameters

```
host:              localhost  (or rabbit service hostname)
port:              5672
vhost:             lms                  (RABBITMQ_DEFAULT_VHOST)
username:          lms_user             (RABBITMQ_DEFAULT_USER)
password:          <from .env>          (RABBITMQ_DEFAULT_PASS)
prefetch_count:    1
ack_mode:          manual
heartbeat:         60
connection_attempts: 3
retry_delay:       5
```

## Tear Down

```bash
# Stop, keep volumes
docker compose down

# Stop and wipe all data
docker compose down -v
```

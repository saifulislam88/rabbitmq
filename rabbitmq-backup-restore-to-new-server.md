### RabbitMQ Backup and Restore to a New Server

This summary outlines the **types of backup methods**, their **dependencies**, **common issues**, and **recommended solutions** for restoring RabbitMQ to a new server — especially when message data and authentication must be retained.

---

#### Backup Methods

##### 1. **Mnesia Backup (Full Internal State)**
- Includes: Users, permissions, queues, exchanges, policies, **durable messages**
- Location: `/var/lib/rabbitmq/mnesia`
- Method:
  ```bash
  tar czf rabbitmq_backup.tar.gz /var/lib/rabbitmq/mnesia
  ```

##### 2. **Export Definitions (Configuration Only)**
- Includes: Users, permissions, queues, exchanges, policies (✅ but **no messages**)
- Command:
  ```bash
  rabbitmqctl export_definitions /tmp/definitions.json
  ```

---

#### 🔗 Dependencies

##### For **Mnesia backup/restore**:
- **RabbitMQ node name**: Must be the same as the original (`rabbit@old-hostname`)
- **Same RabbitMQ version**
- **Same Erlang version** (recommended)
- **Proper ownership and permissions** on the Mnesia files
- **Docker/host config** must support overriding hostname

---

#### ❌ Common Issues

##### 1. **Authentication or user not found errors**
- Caused by node name mismatch.
- Users exist in Mnesia under `rabbit@old-hostname` but RabbitMQ runs as `rabbit@new-hostname`.

##### 2. **Queues and messages not visible**
- Same root cause: Mnesia associates queues/messages with the original node name.
- RabbitMQ silently ignores these entries under mismatched node name.

##### 3. **RabbitMQ starts with a fresh state**
- Happens when restored Mnesia data is not recognized.

---

#### ✅ Solutions

##### ✅ Option A: **Preserve Original Node Name**
- Best for: Full restoration with messages and auth.
- Use:
  ```yaml
  hostname: any-hostname
  environment:
    - RABBITMQ_NODENAME=rabbit@old-hostname
  ```
- Works in Docker, VM, or bare metal.
- Ensures all Mnesia data is valid and fully restored.
- Set the Same Node Name on the New RabbitMQ (Recommended for Simplicity)

```sh
docker-compose.yml
```

```sh
version: '3'
services:
  rabbitmq:
    image: rabbitmq:3.12-management
    hostname: old-hostname
    container_name: rabbitmq
    environment:
      - RABBITMQ_NODENAME=rabbit@old-hostname
    volumes:
      - ./mnesia:/var/lib/rabbitmq/mnesia
      - ./config:/etc/rabbitmq
    ports:
      - 15672:15672
      - 5672:5672
```

##### ⚠️ Option B: **Export and Import Definitions**
- Best for: Moving configuration only.
- Does NOT preserve messages or queue states.
- Steps:
  ```bash
  rabbitmqctl export_definitions /tmp/all.json
  # on new server:
  rabbitmqctl import_definitions /tmp/all.json
  ```

##### ❌ Option C: **Mnesia Hacking or Manual Node Renaming**
- Not supported.
- Risk of corruption.
- Binary format makes editing difficult and error-prone.

---

#### 🔄 Recommended Process (Preserving Messages)

##### 1. **On Source Server**:
```bash
rabbitmqctl stop_app
rabbitmqctl stop
tar czf rabbitmq_backup.tar.gz /var/lib/rabbitmq/mnesia
```

##### 2. **On New Server (Docker or other)**:
- Set the same node name:
  ```yaml
  hostname: old-hostname
  environment:
    - RABBITMQ_NODENAME=rabbit@old-hostname
  ```

- Restore backup:
  ```bash
  tar xzf rabbitmq_backup.tar.gz -C /var/lib/rabbitmq/mnesia
  ```

- Start RabbitMQ and verify:
  ```bash
  rabbitmqctl status
  rabbitmqctl list_users
  rabbitmqctl list_queues
  ```

---

#### 🐍 RabbitMQ Backup and Restore using Python + `pika`

This method uses a Python script with the `pika` library to **programmatically read and write messages** to/from RabbitMQ queues.

---

##### 🔧 Components

- `pika`: Python client for RabbitMQ.
- Script logic:
  - **Backup script**:
    - Connect to queues.
    - Fetch and decode messages (optionally with headers, timestamps).
    - Save to JSON or line-based file.
  - **Restore script**:
    - Read message file.
    - Republish to queues on target RabbitMQ instance.

---

##### ⚠️ Key Issues You Faced

1. **Queue Message Acknowledgement Handling**
2. **Multiline Message Parsing**
3. **Header Preservation**
4. **Queue Declaration Conflicts on Restore**

---

##### ✅ Working Process

###### 🔄 Backup Script
- Connect to `source` RabbitMQ (any hostname).
- Loop through each message in specified queues.
- Store:
  - `queue name`
  - `body`
  - `properties` (headers, timestamp, etc.)
- Output: `messages_backup.jsonl`

###### 🔄 Restore Script
- Connect to `target` RabbitMQ (new host, different node name OK).
- Read each message from file.
- Re-publish using:
  ```python
  channel.basic_publish(exchange='',
                        routing_key=queue,
                        body=message,
                        properties=restored_properties)
  ```

---

##### 🔗 Dependencies

- `pika>=1.3.0`
- Python 3.x
- Both source and destination RabbitMQ servers must:
  - Be reachable from script.
  - Have queues declared (or allow declaring on the fly).
  - Consistent queue configuration (durable, auto-delete flags) if declared again.

---

##### ✅ Benefits

- ✅ Works across RabbitMQ servers with **different hostnames**
- ✅ **Portable** (JSON, text-based)
- ✅ Enables **selective queue migration**
- ✅ Useful for **cross-version migration**

---

##### ⚠️ Limitations

- ❌ Messages in-flight or very large queues may take time
- ❌ No built-in support for message deduplication
- ❌ Ordering is preserved per queue but not guaranteed across queues

---

#### 🔄 Comparison with Mnesia Backup

| Feature                  | Mnesia Backup         | Python `pika` Script           |
|--------------------------|------------------------|--------------------------------|
| Preserves messages       | ✅ Yes                 | ✅ Yes                         |
| Preserves users/auth     | ✅ Yes                 | ❌ No                          |
| Handles hostname change  | ❌ No (must match)     | ✅ Yes                         |
| Queue properties         | ✅ Yes (internal)      | ⚠️ Yes (must redeclare or match) |
| Format                   | Binary (Erlang/Mnesia) | Text/JSON                     |
| Good for selective copy  | ❌ No                  | ✅ Yes                         |

---

#### ✅ Final Recommendations

- Use **Mnesia backup** if:
  - You can preserve `rabbit@hostname`
  - You need a full restore with no hostname change

- Use **Python script with `pika`** if:
  - You’re changing hostnames
  - You want granular control
  - You don’t need user/auth migration
  - You’re okay recreating users and policies manually

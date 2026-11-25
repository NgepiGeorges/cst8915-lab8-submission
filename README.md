# CST8915 Lab 8 – Algonquin Pet Store (On Steroids) on AKS

- **Student:** Georges Ndongmo Ngepi
- **Course:** CST8915 Full-stack Cloud-native Development

## Demo video

YouTube (unlisted): PASTE_YOUTUBE_LINK_HERE

The video shows the deployment of the Algonquin Pet Store with my updated `aps-all-in-one-Task1.yaml` (my own Docker Hub images) on Azure Kubernetes Service, and the application working.

## Files in this repository

| File | Purpose |
| --- | --- |
| `aps-all-in-one-Task1.yaml` | Updated manifest: the six application services use my own images (`ngepigeorges/...:lab8`) |
| `admin-tasks.yaml` | Virtual Customer and Virtual Worker, using my own images |
| `README.md` | Video link, Task 1 summary and Task 2 findings |

`secrets.yaml` (the OpenAI API keys from Brightspace) is **not** in this repository. It is listed in `.gitignore` and was only applied to the cluster with `kubectl apply -f secrets.yaml`.

## Environment

- AKS, Free tier, **2 nodes** `Standard_D2s_v6`, region **Sweden Central**
  (my subscription cannot use the usual VM sizes in Canada Central)
- Images built with Docker Desktop on Windows and pushed to Docker Hub (`ngepigeorges`)
- AI service: direct OpenAI path (`USE_AZURE_OPENAI = "False"`) with the class credentials from Brightspace
- All Azure resources were deleted after recording

## Task 1 – My forks and Docker images

| Service | Language | My fork | My image |
| --- | --- | --- | --- |
| store-front | Vue.js | https://github.com/NgepiGeorges/store-front-L8 | `ngepigeorges/store-front:lab8` |
| store-admin | Vue.js | https://github.com/NgepiGeorges/store-admin-L8 | `ngepigeorges/store-admin:lab8` |
| order-service | Node.js | https://github.com/NgepiGeorges/order-service-L8 | `ngepigeorges/order-service:lab8` |
| product-service | Rust | https://github.com/NgepiGeorges/product-service-L8 | `ngepigeorges/product-service:lab8` |
| makeline-service | Go | https://github.com/NgepiGeorges/makeline-service-L8 | `ngepigeorges/makeline-service:lab8` |
| ai-service | Python | https://github.com/NgepiGeorges/ai-service-L8 | `ngepigeorges/ai-service:lab8` |
| virtual-customer | Rust | https://github.com/NgepiGeorges/virtual-customer-L8 | `ngepigeorges/virtual-customer:lab8` |
| virtual-worker | Rust | https://github.com/NgepiGeorges/virtual-worker-L8 | `ngepigeorges/virtual-worker:lab8` |

Steps I followed: fork → clone → `docker build -t ngepigeorges/<service>:lab8 .` → `docker push` → replace `ramymohamed/` with `ngepigeorges/` in both manifests → `kubectl apply` → verify the running image names with `kubectl get pods -o jsonpath=...`.

## Task 2 – Azure managed services to replace MongoDB and RabbitMQ

In this lab, MongoDB and RabbitMQ run **inside the cluster** as StatefulSets. We must install, update, back up, secure and scale them ourselves. Also, the two StatefulSets in `aps-all-in-one.yaml` have **no `volumeClaimTemplates`**, so they have no persistent volume. A StatefulSet gives a stable name (`mongodb-0`, `rabbitmq-0`), but without a PersistentVolumeClaim the data is still lost when the Pod is recreated (the same problem I showed with RabbitMQ in Lab 7). Managed services move this work and this risk to Azure.

### 1. MongoDB → Azure Cosmos DB for MongoDB

**What it is:** a fully managed NoSQL database service in Azure that supports the **MongoDB API** (wire protocol). It exists in two models: **vCore** (cluster-based, closest to a normal MongoDB server) and **RU-based** (serverless or provisioned throughput).

**Purpose in our app:** store the orders (`orderdb` database, `orders` collection) used by `makeline-service`.

**Why it is a good fit:**
- **Compatible:** `makeline-service` already uses a MongoDB driver. In most cases we only change the connection string (`ORDER_DB_URI`) and add credentials, not the code.
- **Durable by design:** data is stored on Azure-managed, replicated storage with automatic backups. Deleting a Pod or a node in AKS can no longer delete the orders.
- **Less operations work:** Azure handles patching, high availability (SLA), backups and scaling.
- **Security:** encryption at rest, TLS, private endpoints and Microsoft Entra ID integration.

**Problem it solves:** the self-hosted MongoDB in this lab is a single Pod with no persistent volume. It is a single point of failure, and the orders can be lost. Cosmos DB removes both problems.

**Trade-offs:** it costs money (per vCore or per Request Unit), some MongoDB features or versions can differ, and the app becomes dependent on Azure.

### 2. RabbitMQ → Azure Service Bus

**What it is:** a fully managed **enterprise message broker** (PaaS) with **queues** and **publish/subscribe topics**. It uses the **AMQP 1.0** protocol.

**Purpose in our app:** the `orders` queue between `order-service` (producer) and `makeline-service` (consumer).

**Why it is a good fit:**
- **Same pattern:** a durable queue with at-least-once delivery, message locks (peek-lock), dead-letter queues and duplicate detection, which fits the order workflow.
- **Protocol:** the lab already enables the `rabbitmq_amqp1_0` plugin in `config-maps.yaml`, so the services can speak AMQP 1.0, the protocol Service Bus uses. Moving to Service Bus should be mostly configuration (host name and credentials), with some testing.
- **Durable and highly available:** messages are stored by Azure, not inside a Pod. Restarting the cluster cannot lose waiting orders.
- **Less operations work:** no StatefulSet, no plugins, no credentials in plain text in a YAML file (we can use Microsoft Entra ID / managed identity).

**Problem it solves:** the self-hosted RabbitMQ is a single Pod without a persistent volume. If it restarts, the orders waiting in the queue are lost, and nobody is warned.

**Trade-offs:** cost per operation or per tier, vendor lock-in, and it cannot run locally in Docker Compose like RabbitMQ (we would need a cloud namespace or the emulator).

### Other options considered

| Self-hosted | Managed alternative | Note |
| --- | --- | --- |
| MongoDB | Azure Cosmos DB for MongoDB (vCore) | Best compatibility with existing MongoDB code |
| RabbitMQ | Azure Storage Queues | Cheaper and simpler, but fewer features (no topics, no dead-letter queue built in) |
| RabbitMQ | Azure Event Hubs | Built for high-volume event streaming, not for a simple order queue |

### Summary

| Self-hosted in AKS | Azure managed service | Main benefit |
| --- | --- | --- |
| MongoDB StatefulSet | Azure Cosmos DB for MongoDB | Durable, backed-up, highly available order data with almost no code change |
| RabbitMQ StatefulSet | Azure Service Bus | Durable, highly available order queue with no broker to manage |

## Lessons learned

- A StatefulSet alone does not keep data: it also needs a PersistentVolumeClaim.
- Secrets (API keys) must stay out of Git and out of videos. Here they were applied with `kubectl apply -f secrets.yaml` and the file is in `.gitignore`.
- Building Rust images (product-service, virtual-customer, virtual-worker) takes much longer than Node, Python or Go images.
- To control cost, I built and pushed all images on my laptop **before** creating the cluster, and deleted the cluster right after the video.

## References

- Microsoft. (n.d.). *Azure Cosmos DB for MongoDB*. https://learn.microsoft.com/azure/cosmos-db/mongodb/introduction
- Microsoft. (n.d.). *What is Azure Service Bus?* https://learn.microsoft.com/azure/service-bus-messaging/service-bus-messaging-overview
- Microsoft. (n.d.). *AMQP 1.0 in Azure Service Bus and Event Hubs protocol guide*. https://learn.microsoft.com/azure/service-bus-messaging/service-bus-amqp-protocol-guide
- Microsoft. (n.d.). *Storage queues and Service Bus queues – compared and contrasted*. https://learn.microsoft.com/azure/service-bus-messaging/service-bus-azure-and-service-bus-queues-compared-contrasted
- Kubernetes. (n.d.). *StatefulSets*. https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/

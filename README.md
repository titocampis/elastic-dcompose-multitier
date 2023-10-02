# Elasticsearch Multi-tier Cluster using Docker Compose

## 1. Introduction

In order to experiment with a **multi-tier architecture**, we can bring up a docker-compose stack that will initialize three Elasticsearch containers and one Kibana. Each Elasticsearch container will be configured with appropriate node roles so we can simulate a multi-tier with nodes playing different roles, as follow:
- **Node es01:** master, data_content, data_hot
- **Node es02:** master, data_warm
- **Node es03:** master, data_cold
- **Node es04:** data_frozen

## 2. Requirements

Docker compose is required, so in my case I'm using [docker desktop](https://www.docker.com/products/docker-desktop/) which already incorporates docker compose.

> :paperclip: If you’re using Docker Desktop, Docker Compose is installed automatically. Make sure to allocate at least 4GB of memory to Docker Desktop. You can adjust memory usage in Docker Desktop by going to **Settings > Resources**.

## 3. Implementation

### 3.1 docker-compose.yaml

Take a look at file [docker-compose.yaml](docker-compose.yaml).

Notice that for each Elasticsearch container there is an environment variable set that defines which role the Elasticsearch service should play. In order not to have to bring up dedicated master eligible nodes (services) all services play a data role as well as the master role. In a real scenario you’d probably want to have dedicated master eligible nodes and separated data nodes (which will definitely have a stronger hardware resources profile in order to fulfill search queries).

### 3.2 Elasticsearch Cluster creation

Execute Elastisearch Cluster using docker-compose:
```bash
docker-compose up -d
```

>:paperclip: To stop and remove containers created by docker-compose:
> ```bash
> doker-compose down
> ```
> And to remove all resoures created by docker-compose
> ```bash
> doker-compose down -v
> ```

Here is a diagram showing what our cluster looks like:

![elastic_cluster](/pictures/Screenshot%202023-10-02%20091221.png)

One extra step we have to take is to change ownership of the snapshots directory. Docker-compose will mount it as root, but since the Elasticsearch service runs as the elasticsearch user it won’t have access to write in that directory. So, everytime you bring up the stack for the first time you’ll have to run the following command:
```bash
for service in es01 es02 es03 es04; do docker exec $service chown elasticsearch:root /usr/share/elasticsearch/snapshots/; done
```

If everything went fine, you’ll be able to access a Kibana instance by going to the following url in your browser:

```
http://localhost:5601
```

Or in the terminal by:
```
curl -XGET http://localhost:9200/_cluster/health?pretty
```
# Elasticsearch Multi-tier Cluster using Docker Compose

## 1. Introduction

In order to experiment with a **multi-tier architecture**, we can bring up a docker-compose stack that will initialize three Elasticsearch containers and one Kibana. Each Elasticsearch container will be configured with appropriate node roles so we can simulate a multi-tier with nodes playing different roles, as follow:
- **Node es01:** master, data_content, data_hot
- **Node es02:** master, data_warm
- **Node es03:** master, data_cold
- **Node es04:** data_frozen

## 2. Requirements

In my case I'm using [Docker Desktop](https://www.docker.com/products/docker-desktop/), managed from Windows Subsystem for Linux (Ubuntu-22.04 LTS), which incorporates Docker Compose.

> :warning: WARNING: Make sure to allocate at least 4GB of memory to Docker Desktop. You can adjust memory usage in Docker Desktop by going to **Settings > Resources**.

> :warning: WARNING2: The `vm.max_map_count` kernel setting must be set to at least `262144`, if not, it can cause problems.
>
>- To set it just in your current terminal thread execute the following command:
>```bash
>vm.max_map_count=262144
>```
>- To set it permanently on you Linux System add the following line at the end of the file `/etc/sysctl.conf`
>```bash
>vm.max_map_count=262144
>```

## 3. Cluster Creation

### 3.1 docker-compose.yaml

Take a look at file [docker-compose.yaml](docker-compose.yaml).

Notice that for each Elasticsearch container there is an environment variable set that defines which role the Elasticsearch service should play. In order not to have to bring up dedicated master eligible nodes (services) all services play a data role as well as the master role. In a real scenario you’d probably want to have dedicated master eligible nodes and separated data nodes (which will definitely have a stronger hardware resources profile in order to fulfill search queries).

### 3.2 Elasticsearch Cluster creation

Execute Elastisearch Cluster using docker-compose:
```bash
docker-compose up -d
```

>:paperclip: NOTE: To stop and remove containers created by docker-compose:
> ```bash
> doker-compose down
> ```
> To delete the network, containers, and volumes when you stop the cluster, specify the -v option:
> ```bash
> doker-compose down -v
> ```

Here is a diagram showing what our cluster looks like:

![elastic_cluster](/pictures/elastic_cluster.png)

One extra step we have to take is to change ownership of the snapshots directory. Docker-compose will mount it as root, but since the Elasticsearch service runs as the elasticsearch user it won’t have access to write in that directory. So, everytime you bring up the stack for the first time you’ll have to run the following command:
```bash
for service in es01 es02 es03 es04; do docker exec $service chown elasticsearch:root /usr/share/elasticsearch/snapshots/; done
```

To test everything is running OK you can:

- After the cluster has started, open Kibana by going to http://localhost:5601 in a web browser to access Kibana.
- After the cluster has started, execute in some terminal:
```bash
curl -XGET http://localhost:9200/_cluster/health?pretty
```

You can also check the containers are running:
```bash
docker ps
```

## 4. Cluster Configuration

### 4.1 Setting up searchable snapshots and a repository

With our stack up and running we’re almost ready to experiment with the multi-tier architecture. But first we’ll need to set up a couple of things. The first thing we need to take care of is starting the [trial license](https://www.elastic.co/guide/en/elasticsearch/reference/current/start-trial.html), which is necessary because we are going to use the searchable snapshots feature in the frozen tier of our multi-tier architecture and this is not included in the basic license, to know what is included on each licence, you can check the [Official Documentation](https://elastic.co/subscriptions). 

Open the **Kibana Dev Tool** and run the following command in order to start a trial period of 30 days that will give you access to all Elasticsearch premium features (you can also do it using Postman or a shell):

```
POST _license/start_trial?acknowledge=true
 
Response:
{
  "acknowledged" : true,
  "trial_was_started" : true,
  "type" : "trial"
}
```

You can also run the same command using `curl` or `Postman`:
```bash
curl -X POST http://localhost:9200/_license/start_trial?acknowledge=true&pretty
```

### 4.2 Setting up Snapshot Repository

Elasticsearch stores snapshots in an off-cluster storage location called a snapshot repository. Before
you can take or restore snapshots,you must register a snapshot repository on the cluster. The
repository needs to be registered using the `_snapshot` endpoint. In this case we are going to use a **local File System Repository** pointing to the `path.repo` we have configured for our nodes. 

```json
PUT _snapshot/orders-snapshots-repository
{
  "type": "fs",
  "settings": {
    "location": "/usr/share/elasticsearch/snapshots",
    "compress": true
  }
}
```
Response:
```json
{
  "acknowledged" : true
}
```

> :warning: WARNING: If you get a status 500 response saying that the path is not accessible, go back to the previous section and make sure that you’ve run the command that changes ownership of the snapshots directory inside the containers.

## 5. Demonstrating how a multi-tier architecture works

Now we can start experimenting with the multi-tier architecture we created. Our main goal is to watch data migrate from tier to tier and we’ll do that manually.

You can verify that all of our nodes have their roles properly configured by issuing the following command:

```bash
GET _cat/nodes?v&h=name,node.role&s=name
```
Response:
```bash
name node.role
es01 hms
es02 mw
es03 cm
es04 fm
```

You’ll get a response listing our nodes and all the roles each of them play in our cluster:

- **m:** master node
- **s:** content tier
- **h:** hot data tier
- **w:** warm data tier
- **c:** cold data tier
- **f:** frozen data tier

### 5.1 Migrating data through tiers manually (the hard way)

to know how to do it, follow the [documentation](https://opster.com/guides/elasticsearch/capacity-planning/elasticsearch-hot-warm-cold-frozen-architecture/)

### 5.2 Migrating data through tiers automatically (using ILM)

This approach is heavily based on [Index Lifecycle Management](https://opster.com/guides/high-availability/index-lifecycle-policy-management/) (ILM). ILM organizes the lifecycle of your indices in five phases and lets you configure both the transition criteria between phases and a of actions that are to be executed when the index reaches that phase.

Currently you can set up an ILM policy composed of the following five phases:

- **Hot:** The index is actively being updated and queried.
- **Warm:** The index is no longer being updated but is still being queried.
- **Cold:** The index is no longer being updated and is queried infrequently. The information still needs to be searchable, but it’s okay if those queries are slower.
- **Frozen:** The index is no longer being updated and is queried rarely. The information still needs to be searchable, but it’s okay if those queries are extremely slow.
- **Delete:** The index is no longer needed and can safely be removed.

For more information, follow the [documentation](https://opster.com/guides/elasticsearch/capacity-planning/elasticsearch-hot-warm-cold-frozen-architecture/)
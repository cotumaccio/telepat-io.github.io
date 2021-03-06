# Installing and running Telepat

### Repositories guide

*   **[Telepat models](https://github.com/telepat-io/telepat-models)**

    The Telepat models library holds core functionality, and exposes a method API that other components (like the endpoint, and the services) can use to interact with the system. Unless developing the core or working on additional services, you sholdn't need to work with this library directly.

    _Available on npm as 'telepat-models'._

*   **[Telepat API](https://github.com/telepat-io/telepat-api)**

    Built on top of [Express](http://expressjs.com/), the API provides web endpoints that allow interacting with Telepat from web and mobile applications. This is a required part of the Telepat stack.

    _Available on npm as 'telepat-api'._

*   **[Telepat services](https://github.com/telepat-io/telepat-worker)**

    This repo is a container for all the services that come pre-packaged with Telepat:

    *   The aggregation service, that actively monitors data changes and generates patches;
    *   The persistence service, that stores updates in the datastore and sends them off to synchronization;
    *   The transport manager service, that prepares synchronization messages and relays them to each transport type
    *   The synchronization services (currently APN, GCM and Socket.io), handling transport for individual platforms.

    The aggregation and persistence services are required parts of the Telepat stack. If no synchronization service is started, Telepat will act like a traditional, non-real-time API platform and simply deliver data snapshots on-demand.

    _Available on npm as 'telepat-worker'._

*   **[Telepat CLI](https://github.com/telepat-io/telepat-cli)**

    A command-line interface that helps with configuring a Telepat instance and managing applications.

    _Available on npm as 'telepat-cli'._

*   **[Telepat Docker files](https://github.com/telepat-io/telepat-docker-compose-files)**

    A set of Docker Compose files for accelerating the deployment of Telepat as well as its dependencies.

*   **Telepat clients**

    Native clients are available for multiple platforms:

    *   [Javascript client](https://github.com/telepat-io/telepat-js), available on npm and bower as 'telepat-js'
    *   [Android client](https://github.com/telepat-io/telepat-android-sdk), available as a Maven dependency
    *   [iOS client](https://github.com/telepat-io/telepat-ios-sdk), available on CocoaPods as 'Telepat'

### Launching dependencies

As of 0.2.8, Telepat requires 3 external dependencies:

*   A messaging broker. Adapters are provided out of the box for [RabbitMQ](https://www.rabbitmq.com/) and [Kafka](http://kafka.apache.org/).
*   A JSON datastore. An adapter for [Elasticsearch](https://www.elastic.co/) is provided.
*   A [Redis](http://redis.io/) instance to hold Telepat state and configuration data.

You can find installation instructions for all components on their respective website.

Alternatively, we provide Docker Compose files to accelerate deployment. The recipes are separated in two components, shared dependencies and the actual Telepat software. Once you have [docker](https://docs.docker.com/installation/) and [docker-compose](https://docs.docker.com/compose/install/) installed on your machine, here are the steps to get dependencies running:

```bash
git clone https://github.com/telepat-io/telepat-docker-compose-files
cd telepat-docker-compose-files/shared
sudo docker-compose up
```

This will start up all the infrastructure components.

### Configuring dependencies

Telepat needs to create an Elasticsearch index and a series of mappings next. To help you to this, as well as other management tasks, there's a npm package that you can install:

```bash
npm install telepat-cli

# Run these 2 if Docker is not running on localhost
# Mac or Windows, for example
telepat set elasticsearch_host ES_HOST 
telepat set elasticsearch_port ES_PORT

telepat configure elasticsearch
```

The default hostname is locahost, and the default port is 9200\. If running via docker-machine, you can get the host ip by running `docker-machine ip default`.

### Install with Docker

Next, you need to launch the Telepat API and all the other services:

```bash
cd telepat-docker-compose-files/telepat
sudo docker-compose up
```

Right now everything should be up and running. The API instance is available on the same IP as your docker machine.

The default ports are 3000 for the API and 80 for the websocket service.

### Install from GitHub

The Telepat backend stack is made up of two components, that will each need configuration when installed from source:

*   The API ([https://github.com/telepat-io/telepat-api](https://github.com/telepat-io/telepat-api)). To start this, simply run

    ```bash
    ./bin/www
    ```

You can also set the PORT environment variable to make the API listen on a port different than the default 3000.

*   The services ([https://github.com/telepat-io/telepat-worker](https://github.com/telepat-io/telepat-worker)). To start, run

    ```bash
    node index.js -t topic_name -i worker_index
    ```

    For built-in services, these are the commands to start up:

    *   node telepat-worker/index.js -t aggregation -i 0
    *   node telepat-worker/index.js -t write -i 0
    *   node telepat-worker/index.js -t transport_manager -i 0
    *   node telepat-worker/index.js -t android_transport -i 0
    *   node telepat-worker/index.js -t ios_transport -i 0
    *   node telepat-worker/index.js -t sockets_transport -i 0

To configure the components, create a 'config.json' file in the root of each directory. You can start from the 'config.example.json' template and fill in your specific parameters / remove configurations for services not in use.

### Configuration

**Scaling considerations**

Depending on your specific workload and concurrency situations, you will find that some components, such as the API endpoint or the sockets transport service need to be scaled out. You can start any number of instances of the same service type, on the same or different machines. For the service side, increment the value of the -i (index) parameter for each worker. Components that communicate directly with the clients (the API and sockets service) can be placed behind a load balancer (typically HAProxy has been used successfully with production deployments, using a source load balancing algorithm for socket.io load balancing).

**config.json fields**

*   main_database: The adapter type that will be used for the persistent object storage (use "ElasticSearch" to use an ElasticSearch cluster for storage)
*   message_queue: The adapter type that will be used for the message queue system (use "amqp" or "kafka")
*   logger: Telepat uses [Winston](https://github.com/winstonjs/winston) for logging activity and error data. Place a Winston configuration object on this key in order to configure logging. Example:

    ```json
    "logger": {
        "type": "Console",
        "settings": {
          "level": "info"
        }
    };
    ```

*   ElasticSearch: Configuration information for the Elasticsearch adapter. You can specify all the nodes in the cluster using the "hosts" array. If your cluster has auto-discovery configured you can specify a "host" and "port" set of keys and the rest of the cluster nodes will be dynamically found and used. Example:

    ```json
    "ElasticSearch": {
        "hosts": ["172.31.49.149:9200"],
        "index": "default"
    };
    ```

*   redis: Configuration information for the redis instance, used by Telepat to hold subscription and device information. Example:

    ```json
    "redis": {
        "host": "172.31.49.149",
        "port": 6379
    };
    ```

*   redisCache: Configuration information for the redis instance used for data caching (such as count calls results). You can use the same instance as for state information, or a separate one, for distributing load.

    ```json
    "redisCache": {
        "host": "172.31.49.149",
        "port": 6379
    };
    ```

*   login_providers: Configuration information for facebook and twitter login providers.

    ```json
    "login_providers": {
        "facebook": {
          "client_id": "",
          "client_secret": ""
        },
        "twitter": {
          "consumer_key": "",
          "consumer_secret": ""
        }
    };
    ```

**Note**: for configuring this setting on your Telepat Cloud instance, send us a request to [support@telepat.io](mailto:support@telepat.io)

*   amqp: Configuration information for the AMQP queue adapter (RabbitMQ or ActiveMQ). Example:

    ```json
    "amqp": {
        "host": "172.31.49.149",
        "user": "guest",
        "password": "guest"
    };
    ```

*   password_salt: A password salt used by bcrypt to secure user passwords against dictionary attacks. The hash has the following format: `$<id>$<cost>$<salt><digest>` and can be generated in bash using the commands below:

    ```bash
    node
    bcrypt = require('bcrypt')    
    bcrypt.genSaltSync()
    ```

*   mandrill: Telepat uses Mandrill for sending transactional emails, such as account confirmation emails or password reset messages. The API key can be configured similarly to the following example:

    ```json
    "mandrill": {
        "api_key": ""
    };
    ```

**Note**: for configuring this setting on your Telepat Cloud instance, send us a request to [support@telepat.io](mailto:support@telepat.io)

**Environment variables**

Alternatively, you can configure Telepat using environment variables. Here are the environment variables that you can set:

*   TP_KFK_HOST: Kafka (zoekeeper) server
*   TP_KFK_PORT: Kafka (zoekeeper) server port
*   TP_KFK_CLIENT: Name for the kafka client
*   TP_REDIS_HOST: Redis database server
*   TP_REDIS_PORT: Redis server port
*   TP_REDISCACHE:_HOST Redis caching instance hostname / IP address
*   TP_REDISCACHE:_PORT Redis caching instance port
*   TP_PW_SALT: Bcrypt formatted password salt for securing user passwords
*   TP_MAIN_DB: Name of the main database which to use. Should be the same as the exported variable in telepat-models
*   TP_ES_HOST: Elasticsearch server
*   TP_ES_PORT: Elasticsearch server port
*   TP_AMQP_HOST: RabbitMQ host
*   TP_AMQP_USER: RabbitMQ authentication username
*   TP_AMQP_PASSWORD: RabbitMQ authentication password

# Creating an app

After bootup, you need to create a new administration user account - Telepat CLI can help you setup your new app in no time. Here are the steps you need to take to create a new app:

* Register a new admin:

    ```bash
    telepat add admin --email EMAIL --password PASSWORD
    ```

* Create the app:

    ```bash
    telepat add app --name APP_NAME --apiKey API_KEY
    ```

* Create at least one context (as a container for your objects):

    ```bash
    telepat add context --contextName CONTEXT_NAME
    ```

*   Create a schema file, and feed it into your app, so Telepat knows the types of objects you'll be working with. You need to have a schema that defines at the very least the types of objects; you can optionally also add information about object parameter names or relationships.

    ```bash
    telepat set schema --filename PATH_TO_SCHEMA_JSON --apiKey API_KEY
    ```

# A word on stability

Telepat is beta software. Although specific versions of Telepat are stable and running production applications right now, we are still in the stage of making design and interface decisions based on trials and feedback, so please keep in mind:

*   Some functionalities implemented by the backend components may not be yet implemented in specific clients
*   API endpoints specifications might change

We appreciate support from the community in identifying and fixing issues, so if you run into any trouble, please open up an issue on the proper repo and we'll be quick to help out.

# Configuration

## Kelon (CLI)

Kelon already comes with a CLI. Just type `kelon --help` after you installed it.
```
usage: kelon [<flags>] <command> [<args> ...]

Kelon policy enforcer.

Flags:
  -h, --help                   Show context-sensitive help (also try --help-long and --help-man).
  -d, --datastore-conf=./datastore.yml  
                               Path to the datastore configuration yaml.
  -a, --api-conf=./api.yml     Path to the api configuration yaml.
      --config-watcher-path=CONFIG-WATCHER-PATH  
                               Path where the config watcher should listen for changes.
  -o, --opa-conf=./opa.yml     Path to the OPA configuration yaml.
  -r, --rego-dir=REGO-DIR      Dir containing .rego files which will be loaded into OPA.
      --path-prefix="/v1"      Prefix which is used to proxy OPA's Data-API.
  -p, --port=8181              Port on which the proxy endpoint is served.
      --preprocess-policies    Preprocess incoming policies for internal use-case (EXPERIMENTAL FEATURE! DO NOT USE!).
      --log-level=INFO         Log-Level for Kelon. Must be one of [DEBUG, INFO, WARN, ERROR]
      --access-decision-log-level=ALL
                               Configuration for logging access decisions while Info-Log-Level for Kelon. Must be one of [ALL, ALLOW, DENY, NONE]
      --respond-with-status-code  
                               Communicate Decision via status code 200 (ALLOW) or 403 (DENY).
      --telemetry-service=TELEMETRY-SERVICE  
                               Service that is used for telemetry [Prometheus]
      --version                Show application version.

Commands:
  help [<command>...]
    Show help.

  run
    Run kelon in production mode.

```

In addition to that Kelon provides the possibility to be configured via environment variables. This may be handy if you want to run it inside a container.

### General

|Flag|Short|Environment|Default|Type|
|----|-----|-----------|-------|----|
|--datastore-conf|-d|DATASTORE_CONF|./datastore.yml|Existing File|
|--api-conf|-a|API_CONF|./api.yml|Existing File|
|--config-watcher-path||CONFIG_WATCHER_PATH|./policies|Existing Dir|
|--opa-conf|-o|OPA_CONF|./opa.yml|Existing File|
|--rego-dir|-r|REGO_DIR|./policies|Existing Dir|
|--path-prefix||PATH_PREFIX|/v1|String|
|--port|-p|PORT|8181|Number|
|--preprocess-policies||PREPROCESS_POLICIES|false|Boolean|
|--respond-with-status-code||RESPOND_WITH_STATUS_CODE|false|Boolean|
|--access-decision-log-level||ACCESS_DECISION_LOG_LEVEL|ALL|Enum|

### Telemetry

|Flag|Short|Environment|Default|Type|
|----|-----|-----------|-------|----|
|--telemetry-service||TELEMETRY_SERVICE||Enum|

## datastore.yml

The `datastore.yml` defines all available datastores and their entities.

```yaml
# List of all datastores which kelon should be able to connect to
# Each datastore has to have an entity schema configured!
datastores:

  # Datastore's alias (it will be used throughout all your policies).
  mysql:
  
    # Datastore type
    # Kelon supports [mysql, postgres, mongo]
    [type: <string>]

    # Properties that are needed to connect to the datastore
    connection:     
    
      # Hostname or IP of the database
      [host: <string>]

      # Port under which the database is available
      [port: <int>]

      # Name of the database to connect to
      # Many database servers run multiple databases.
      # In this case create one datastore for each database
      [database: <string>]

      # The user which is used for the database connection
      [user: <string>]

      # The password of the user
      [password: <string>]

      # Connection options for the database connection
      # To get all available options for your database, see section "Connection options" below
      [<option>: <string>]

    # Metadata attached to the datastore
    metadata:

      # Only used from SQL-Datastores
      # Sets max idle connections of the database connection pool
      
      [maxIdleConnections: <int>]

      # Only used from SQL-Datastores
      # Sets max open connections of the database connection pool
      [maxOpenConnections: <int>]

      # Only used from SQL-Datastores
      # Sets max lifetime of each connection inside the database connection pool
      [connectionMaxLifetimeSeconds: <int>]


# Entity-Schemas define the entities of one schema inside a datastore.
# An entity_schema has several purposes:
#   => Help kelon to detect misspellings in regos
#   => Create entity-aliases for complex table-names
#   => Define the structure of nested entities (Just for No-SQL DBs)
entity_schemas:
  
  # Datastore's alias
  # This example shows how each SQL-Datastore should be configured.
  mysql:

    # Schema name
    # Kelon automatically adds the schema name to entities
    # This means that 'data.mysql.users' inside a rego will be translated 
    # to 'appstore.users' in the resulting SQL-Statement.
    #
    # !!! The entities inside all schemas of one Datastore have to be disjunct !!!
    # Otherwise kelon will fail on startup
    appstore:

      # List of all entities of schema 'appstore'
      entities:

        # Entity with the name inside the datastore
        - [name: <string>]

          # And an optional alias (used inside regos)
          [alias: <string> | optional]
  

  # This example shows how each No-SQL-Datastore should be configured.
  mongo:

    # Schema name
    # Because No-SQL-Datastores don't have concept for schemas, 
    # Kelon ignores the schema name for i.e. a MongoDB.
    #
    # !!! The entities inside all schemas of one Datastore have to be disjunct !!!
    # Otherwise kelon will fail on startup
    appstore:

      # List of all entities (in case of MongoDB each entity is a collection)
      entities:

        # Collection with it's name inside the datastore
        - [name: <string>]

          # Collection's optinal alias (used inside regos)
          [alias: <string> | optional]

          # Nested entities (Only supported for No-SQL-Datastores)
          # The depth of nesting entities is unlimited.
          # It makes no difference if the nested entities are arrays or objects
          entities:

            # Name of nested entity
            - [name: <string>]
              # Collection's optinal alias (used inside regos)
              [alias: <string> | optional]
```

### Connection options

Kelon supports different databases (PostgreSQL, MySQL and MongoDB) each of them having different connection options.
To keep the configuration of connection options as simple as possible, Kelon just passes all key-value-pairs of each the datastore connection (despite the dedicated ones like i.e. username, host, port, etc.) directly to the used database adapter. Following table should help to lookup all available options for each supported database:

|Database|Used driver|Options|
|--------|-----------|-------|
|PostgreSQL|github.com/lib/pq|[Go Docs, Connection String Parameters](https://godoc.org/github.com/lib/pq#hdr-Connection_String_Parameters) |
|MySQL|github.com/go-sql-driver/mysql|[Go Docs, DSN Parameters](https://github.com/go-sql-driver/mysql#parameters)|
|MongoDB|go.mongodb.org/mongo-driver/mongo|[Mongo-Docs, Connection options](https://docs.mongodb.com/manual/reference/connection-string/#connections-connection-options)|

### Injecting environment variables

Each value inside the datastore.yml file can be replaced with a string follwing the pattern `${<environment variable>}` which is replaced with the value of the environment variable at startup. 

## call-operands/{datastore-type}.yml

Open Policy Agent has [builtin call operands](https://www.openpolicyagent.org/docs/latest/policy-language/#operators) which may not be the same as the ones of each datastore.
Therefore each datastore-type has to have a mapping-file inside the folder 'call-operands'. The name of each file inside this directory has to match the type of the datastore.

```yaml
# Call operands map OPA's functions to datastore-native ones.
# You find more advanced files in directory /call-operands
call-operands:
  
  # Operand name inside OPA's context
  - [op: <string>]

    # Number of arguments the function call should take
    # Used for validation
    [args: <int>]

    # Mapping which mappes the OPA-native call to a Datastore-native call.
    # This is done by defining a pattern where all occurences of $<arg-pos>
    # are replaced with the original argument at position <arg-pos> (zero based).
    [mapping: <string>]
```

## api.yml

The `datastore.yml` contains all necessary configuration kelon needs to map incoming api-requests to OPA-Queries executed inside datastores.

```yaml
# Mappings for the APIs that your services expose
apis:
  
  # API-Collection
  # Each collection can have a path prefix which is prepended to every API-Mapping
  # inside the collection.
  - [path-prefix: <URL|string>]

    # All rego-queries mapped by this API-collection will include 
    # this datastore-alias as unknown.
    # This means that the regos this collection targets can contain
    # something like 'data.<datastore-alias>.<entity>.<attribute>'
    [datastore: <datastore-alias|string>]

    # Path-Mappings for incoming paths
    # (the most specific mapping is picked in case of multiple mappings)
    mappings:

      # Path-Mapping
      # Each mapping has to specify a path (regex) and a package which is used
      # by kelon to generate the target package of an mapped query.
      - [path: <regex|string>] 
        
        # OPA's target package of this Path-Mapping
        [package: <string>]
        
        # HTTP-Methods this path should map (optional)
        # If no methods are specified, kellon will match [GET,PUT,POST,DELETE,PATCH]
        methods:
          - [<string> ...]
```


## opa.yml

Internally used by Open Policy Agent. Klick [here](https://www.openpolicyagent.org/docs/latest/configuration/) for more information.

```yaml
labels:
  app: Kelon
  region: europe-west3
  environment: development

decision_logs:
  console: true
  reporting:
    min_delay_seconds: 300
    max_delay_seconds: 600

default_decision: /http/authz/allow
```
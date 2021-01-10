# Redis AUTH and mTLS with Envoy

Sample config to setup Envoy to proxy Redis traffic using mTLS and Redis AUTH.

Both client->envoy->redis is secured by redis AUTH

`client`->`envoy`-->`redis` 

  client->redis is on port `:6000` while envoy->redis is on port `:6379`

the whole system uses mTLS end to end. 

I’m writing this up since i found it really tricky to setup the envoy side of things…especially with both downstream and upstream AUTH:

What we’re going to do:

Setup a go redis client app to talk via TLS to envoy. Envoy will then proxy requests to Redis server.

This git repo is just the git version with the config files of a previous post here [Redis with Envoy](https://blog.salrashid.me/posts/redis_envoy/)


First download docker and golang 1.14:

Docker:  

Note: we are using envoy v3 1.17

```bash
docker cp `docker create envoyproxy/envoy-dev:latest`:/usr/local/bin/envoy .
```

### Setup

Download the source files froe the git repo here. You should end up with:

* `basic.yaml`: Envoy config file
* `CA_crt.epm`: The CA Cert for mtls
* `server.crt`, server.key: Server certs envoy will use
* `client.crt`, client.key: client side certs the go app will use
* `main.go`: Redis golang client that will connect to Envoy


### Redis

Once you donwload redis, edit redis.conf and uncomment the following line to enable default user AUTH:

# IMPORTANT NOTE: starting with Redis 6 "requirepass" is just a compatiblity
# layer on top of the new ACL system. The option effect will be just setting
# the password for the default user. Clients will still authenticate using
# AUTH <password> as usually, or more explicitly with AUTH default <password>
# if they follow the new protocol: both will work.
#
requirepass foobared

Start Redis:

```bash
docker run -v `pwd`/certs:/certs -p 6379:6379 redis \
         --tls-port 6379 --port 0  \
         --tls-cert-file /certs/server.crt    \
         --tls-key-file /certs/server.key     \
         --tls-ca-cert-file /certs/CA_crt.pem \
         --requirepass foobared
```

### Envoy

Start Envoy:

```
envoy  -c basic.yaml  -l debug
```

The whole reason for this article is because i found it hard to configure enovy…so here it is:

Some things to point out below:

* Envoy Listens on mTLS: the config tls_context:

```yaml
      transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
          require_client_certificate: true
          common_tls_context:
            tls_certificates:
            - certificate_chain:
                filename: certs/server.crt
              private_key:
                filename: certs/server.key
            validation_context:
              trusted_ca:
                filename: certs/CA_crt.pem
```

* Clients connecting to envoy must provide a redis password: `bar`

```yaml
    filter_chains:
    - filters:
      - name: envoy.filters.network.redis_proxy
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.redis_proxy.v3.RedisProxy
          stat_prefix: egress_redis
          settings:
            op_timeout: 5s
          prefix_routes:
            catch_all_route:
              cluster: redis_cluster 
          downstream_auth_password:
            inline_string: "bar"
```

* Envoy connect to Redis outbound with mtls: the config transport_socket

```yaml
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        common_tls_context:
          tls_certificates:
            certificate_chain: 
              filename: "certs/client.crt"
            private_key:
              filename: "certs/client.key"       
          validation_context:
            trusted_ca:
              filename: "certs/CA_crt.pem"
```

* Envoy connects to Redis must provide a redis: `foobared`

```yaml
  clusters:
  - name: redis_cluster
    connect_timeout: 1s
    type: strict_dns
    load_assignment:
      cluster_name: redis_cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: 6379
    typed_extension_protocol_options:
      envoy.redis_proxy:
        "@type": type.googleapis.com/google.protobuf.Struct
        value:
          auth_password:
            inline_string: "foobared"
```

In enovy-speak, the client is downstream while redis is upstream as far as envoy is concerned.


### Start go redis client

main.go connects to envoy on port :6000 and presents a client certificate to it:


If you run the app now you’ll see a pong and the value you just saved:

```
$ go run main.go 
Ping Response: PONG
Incrementing key:
key 1
key 2
key 3
key 4
key 5
```

In the envoy logs, you'll see it traverses the filter

```log
[2021-01-10 18:37:22.085][236708][debug][conn_handler] [source/server/connection_handler_impl.cc:503] [C4] new connection
[2021-01-10 18:37:22.088][236708][debug][redis] [source/extensions/filters/network/redis_proxy/command_splitter_impl.cc:539] redis: splitting '["set", "key", "0"]'
[2021-01-10 18:37:22.089][236708][debug][redis] [source/extensions/filters/network/redis_proxy/command_splitter_impl.cc:539] redis: splitting '["incr", "key"]'
[2021-01-10 18:37:22.089][236708][debug][redis] [source/extensions/filters/network/redis_proxy/command_splitter_impl.cc:539] redis: splitting '["get", "key"]'
[2021-01-10 18:37:22.089][236708][debug][redis] [source/extensions/filters/network/redis_proxy/command_splitter_impl.cc:539] redis: splitting '["incr", "key"]'
[2021-01-10 18:37:22.089][236708][debug][redis] [source/extensions/filters/network/redis_proxy/command_splitter_impl.cc:539] redis: splitting '["get", "key"]'
[2021-01-10 18:37:22.090][236708][debug][redis] [source/extensions/filters/network/redis_proxy/command_splitter_impl.cc:539] redis: splitting '["incr", "key"]'
[2021-01-10 18:37:22.090][236708][debug][redis] [source/extensions/filters/network/redis_proxy/command_splitter_impl.cc:539] redis: splitting '["get", "key"]'
[2021-01-10 18:37:22.090][236708][debug][redis] [source/extensions/filters/network/redis_proxy/command_splitter_impl.cc:539] redis: splitting '["incr", "key"]'
[2021-01-10 18:37:22.090][236708][debug][redis] [source/extensions/filters/network/redis_proxy/command_splitter_impl.cc:539] redis: splitting '["get", "key"]'
[2021-01-10 18:37:22.090][236708][debug][redis] [source/extensions/filters/network/redis_proxy/command_splitter_impl.cc:539] redis: splitting '["incr", "key"]'
[2021-01-10 18:37:22.090][236708][debug][redis] [source/extensions/filters/network/redis_proxy/command_splitter_impl.cc:539] redis: splitting '["get", "key"]'
[2021-01-10 18:37:22.091][236708][debug][connection] [source/common/network/connection_impl.cc:619] [C4] remote close

```

…yeah, thats it..
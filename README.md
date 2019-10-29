grpcbox
=====

[![CircleCI](https://circleci.com/gh/tsloughter/grpcbox.svg?style=svg)](https://circleci.com/gh/tsloughter/grpcbox)
[![codecov](https://codecov.io/gh/tsloughter/grpcbox/branch/master/graph/badge.svg)](https://codecov.io/gh/tsloughter/grpcbox)
[![Hex.pm](https://img.shields.io/hexpm/v/grpcbox.svg?maxAge=2592000)](https://hex.pm/packages/grpcbox)
[![Hex.pm](https://img.shields.io/hexpm/dt/grpcbox.svg?maxAge=2592000)](https://hex.pm/packages/grpcbox)

Library for creating [grpc](https://grpc.io) services (client and server) in Erlang, based on the [chatterbox](https://github.com/joedevivo/chatterbox) http2 library.

Features
---

* Unary, client stream, server stream and bidirectional rpcs
* Client load balancing
* Interceptors
* Health check service
* Reflection service
* [OpenCensus](https://opencensus.io) interceptors for stats and tracing
* [Plugin](https://github.com/tsloughter/grpcbox_plugin) for generating clients and behaviour type specs for service server implementation

Implementing a Service Server
----

The quickest way to play around is with the test service and client that is used by `grpcbox`. Simply pull up a shell with, `rebar3 as test shell` and the route guide service will start on port 8080 and you'll have the client, `routeguide_route_guide_client`, in the path.

The easiest way to get started on your own project is using the plugin, [grpcbox_plugin](https://github.com/tsloughter/grpcbox_plugin):

```erlang
{deps, [grpcbox]}.

{grpc, [{protos, "protos"},
        {gpb_opts, [{module_name_suffix, "_pb"}]}]}.

{plugins, [grpcbox_plugin]}.
```

Currently `grpcbox` and the plugin are a bit picky and the `gpb` options will always include `[use_packages, maps, {i, "."}, {o, "src"}]`.

Assuming the `protos` directory of your application has the `route_guide.proto` found in this repo, `protos/route_guide.proto`, the output from running the plugin will be:

```shell
$ rebar3 grpc gen
===> Writing src/route_guide_pb.erl
===> Writing src/grpcbox_route_guide_bhvr.erl
```

A behaviour is used because it provides a way to generate the interface and types without being where the actual implementation is also done. This way if a change happens to the proto you can regenerate the interface without any issues with the implementation of the service, simply then update the implementation callbacks to match the changed interface.

Runtime configuration for `grpcbox` can be done in `sys.config`, specifying the compiled proto modules to use for finding the services available, which services to actually enable for requests and what module implements them, acceptor pool and http server settings. See `interop/config/sys.config` for a working example.

In the interop config the portion for defining services to handle requests for is:

``` erlrang
{grpcbox, [{servers, [#{grpc_opts => #{service_protos => [test_pb],
                                       services => #{'grpc.testing.TestService' => grpc_testing_test_service}}}]},
...
```

`test_pb` is the `gpb` generated module that exports `get_service_names/0`. The results of that function are used to construct the metadata needed for handling requests. The `services` map gives the module to call for handling methods of a service. If a service is not defined in that map it will result in the grpc error code 12, `Unimplemented`.

The services will be started when the application starts assuming the services are all configured in the `sys.config` and it is loaded. To manually start a service use either `grpcbox:start_server/1` which will start a `grpcbox_service_sup` supervisor under the `grpcbox_services_simple_sup` simple one for one supervisor, or get a child spec `grpcbox:server_child_spec(ServerOpts, GrpcOpts, ListenOpts, PoolOpts, TransportOpts)` to include the service supervisor in your own supervision tree.

#### Unary RPC

Unary RPCs receive a single request and return a single response. The RPC `GetFeature` takes a single `Point` and returns the `Feature` at that point:

```protobuf
rpc GetFeature(Point) returns (Feature) {}
```

The callback generated by the `grpcbox_plugin` will look like:

```erlang
-callback get_feature(ctx:t(), route_guide_pb:point()) ->
    {ok, route_guide_pb:feature(), ctx:ctx(} | grpcbox_stream:grpc_error_response().
```

And the implementation is as simple as an Erlang function that takes the arguments `Ctx`, the context of this current request, and a `Point` map, returning a `Feature` map:

```erlang
get_feature(Ctx, Point) ->
    Feature = #{name => find_point(Point, data()),
                location => Point},
    {ok, Feature, Ctx}.
```

#### Streaming Output

Instead of returning a single feature the server can stream a response of multiple features by defining the RPC to have a `stream Feature` return:

```protobuf
rpc ListFeatures(Rectangle) returns (stream Feature) {}
```

In this case the callback still receives a map argument but also a `grpcbox_stream` argument:

```erlang
-callback list_features(route_guide_pb:rectangle(), grpcbox_stream:t()) ->
    ok | {error, term()}.
```

The `GrpcStream` variable is passed to `grpcbox_stream:send/2` for returning an individual feature over the stream to the client. The stream is ended by the server when the function completes.

```erlang
list_features(_Message, GrpcStream) ->
    grpcbox_stream:send(#{name => <<"Tour Eiffel">>,
                                        location => #{latitude => 3,
                                                      longitude => 5}}, GrpcStream),
    grpcbox_stream:send(#{name => <<"Louvre">>,
                          location => #{latitude => 4,
                                        longitude => 5}}, GrpcStream),
    ok.
```

#### Streaming Input

The client can also stream a sequence of messages:

```protobuf
rpc RecordRoute(stream Point) returns (RouteSummary) {}
```

In this case the callback receives a `reference()` instead of a direct value from the client:

```erlang
-callback record_route(reference(), grpcbox_stream:t()) ->
    {ok, route_guide_pb:route_summary()} | {error, term()}.
```

The process the callback is running in will receive the individual messages on the stream as tuples `{reference(), route_guide_pb:point()}`. The end of the stream is sent as the message `{reference(), eos}` at which point the function can return the response:

```erlang
record_route(Ref, GrpcStream) ->
    record_route(Ref, #{t_start => erlang:system_time(1),
                            acc => []}, GrpcStream).

record_route(Ref, Data=#{t_start := T0, acc := Points}, GrpcStream) ->
    receive
        {Ref, eos} ->
            {ok, #{elapsed_time => erlang:system_time(1) - T0,
                   point_count => length(Points),
                   feature_count => count_features(Points),
                   distance => distance(Points)}, GrpcStream};
        {Ref, Point} ->
            record_route(Ref, Data#{acc => [Point | Points]}, GrpcStream)
    end.
```

#### Streaming In and Out

A bidrectional streaming RPC is defined when both input and output are streams:
 
```protobuf
rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
```

```erlang
-callback route_chat(reference(), grpcbox_stream:t()) ->
    ok | {error, term()}.
```

The sequence of input messages will again be sent to the callback's process as Erlang messages and any output messages are sent to the client with `grpcbox_stream`:

```erlang
route_chat(Ref, GrpcStream) ->
    route_chat(Ref, [], GrpcStream).

route_chat(Ref, Data, GrpcStream) ->
    receive
        {Ref, eos} ->
            ok;
        {Ref, #{location := Location} = P} ->
            Messages = proplists:get_all_values(Location, Data),
            [grpcbox_stream:send(Message, GrpcStream) || Message <- Messages],
            route_chat(Ref, [{Location, P} | Data], GrpcStream)
    end.
```

#### Interceptors

##### Unary Interceptor

A unary interceptor can be any function that accepts a context, decoded request body, server info map and the method function:

```erlang
some_unary_interceptor(Ctx, Request, ServerInfo, Fun) ->
    %% do some interception stuff
    Fun(Ctx, Request).
```

The interceptor is configured in the `grpc_opts` set in the environment or passed to the supervisor `start_child` function. An example from the test suite sets `grpc_opts` in the application environment:

```erlang
#{service_protos => [route_guide_pb],
  unary_interceptor => fun(Ctx, Req, _, Method) ->
                         Method(Ctx, #{latitude => 30,
                                       longitude => 90})
                       end}
```

##### Streaming Interceptor

##### Middleware

There is a provided interceptor `grpcbox_chain_interceptor` which accepts a list of interceptors to apply in order, with the final interceptor calling the method handler. An example from the test suite adds a trailer in each interceptor to show the chain working:

```erlang
#{service_protos => [route_guide_pb],
  unary_interceptor =>
    grpcbox_chain_interceptor:unary([fun ?MODULE:one/4,
                                     fun ?MODULE:two/4,
                                     fun ?MODULE:three/4])}
```

#### Tracing

The provided interceptor `grpcbox_trace` supports the [OpenCensus](http://opencensus.io/) wire protocol using [opencensus-erlang](https://github.com/census-instrumentation/opencensus-erlang). It will use the `trace_id`, `span_id` and any options or tags from the trace context.

Configure as an interceptor:

```erlang
#{service_protos => [route_guide_pb],
  unary_interceptor => {grpcbox_trace, unary}}
```

Or as a middleware in the chain interceptor:

```erlang
#{service_protos => [route_guide_pb],
  unary_interceptor =>
    grpcbox_chain_interceptor:unary([..., 
                                     fun grpcbox_trace:unary/4, 
                                     ...])}
```

See [opencensus-erlang](https://github.com/census-instrumentation/opencensus-erlang) for details on configuring reporters.

#### Statistics

Statistics are collected by implementing a stats handler module. A handler for OpenCensus stats (be sure to include [OpenCensus](https://hex.pm/packages/opencensus) as a dependency and make sure it starts on boot) is provided and can be enabled for the server with a config option:

``` erlang
{grpcbox, [{servers, [#{grpc_opts => #{stats_handler => grpcbox_oc_stats_handler
                                       ...}}]}]}
```

For the client the stats handler is a per-channel configuration, see the Defining Channels section below.

You can verify it is working by enabling the stdout exporter:

``` erlang
 {opencensus, [{stat, [{exporters, [{oc_stat_exporter_stdout, []}]}]}]}
```

For actual use, an [exporter for Prometheus](https://github.com/opencensus-beam/prometheus) is available.

Details on all the metrics that are collected can be found in the [OpenCensus gRPC Stats specification](https://github.com/census-instrumentation/opencensus-specs/blob/master/stats/gRPC.md).

#### Metadata

Metadata is sent in headers and trailers.

Using a Service Client
----

For each service in the protos passed to `rebar3 gprc gen` it will genrate a `<service>_client` module containing a function for each method in the service.

#### Defining Channels

Channels maintain connections to grpc servers and offer client side load balancing between servers with various methods, round robin, random, hash.

If no channel is specified in the options to a rpc call the `default_channel` is used. Setting the default to connect to localhost on port 8080 in your `sys.config` would look like:

```
{client, #{channels => [{default_channel, [{http, "localhost", 8080, []}], #{}}]}}
```

The empty map at the end can contain configuration for the load balancing algorithm, interceptors, statistics handling and compression:

```
#{balancer => round_robin | random | hash | direct | claim,
  encoding => identity | gzip | deflate | snappy | atom(),
  stats_handler => grpcbox_oc_stats_handler,
  unary_interceptor => term(),
  stream_interceptor => term()} 
```

The default balancer is round robin and encoding is identity (no compression). Encoding can also be passed in the options map to individual requests.

#### Calling Unary Client RPC

The `RouteGuide` service has a single unary method, `GetFeature`, in the client we have a function `get_feature/2`:

```erlang
Point = #{latitude => 409146138, longitude => -746188906},
{ok, Feature, HeadersAndTrailers} = routeguide_route_guide_client:get_feature(Point).
```

#### Client Streaming RPC

```erlang
{ok, S} = routeguide_route_guide_client:record_route(),
ok = grpcbox_client:send(S, #{latitude => 409146138, longitude => -746188906}),
ok = grpcbox_client:send(S, #{latitude => 234818903, longitude => -823423910}),
ok = grpcbox_client:close_send(S),
{ok, #{point_count := 2} = grpcbox_client:recv_data(S)).
```

#### Client with Server Streaming RPC

```erlang
Rectangle = #{hi => #{latitude => 1, longitude => 2},
              lo => #{latitude => 3, longitude => 5}},
{ok, S} = routeguide_route_guide_client:list_features(Rectangle),
{ok, #{<<":status">> := <<"200">>}} = grpcbox_client:recv_headers(S),
{ok, #{name := _} = grpcbox_client:recv_data(S),
{ok, #{name := _}} = grpcbox_client:recv_data(S),
{ok, _} = grpcbox_client:recv_trailers(S).
```

#### Bidirectional RPC

```erlrang
{ok, S} = routeguide_route_guide_client:route_chat(),
ok = grpcbox_client:send(S, #{location => #{latitude => 1, longitude => 1}, message => <<"hello there">>}),
ok = grpcbox_client:send(S, #{location => #{latitude => 1, longitude => 1}, message => <<"hello there">>}),
{ok, #{message := <<"hello there">>}} = grpcbox_client:recv_data(S)),
ok = grpcbox_client:send(S, #{location => #{latitude => 1, longitude => 1}, message => <<"hello there">>}),
{ok, #{message := <<"hello there">>}}, grpcbox_client:close_and_recv(S)).
```

#### Context

Client calls optionally accept a [context](https://hex.pm/packages/ctx) as the first argument. Contexts are used to set and propagate deadlines and [OpenCensus](https://hex.pm/packages/opencensus) tags.

```erlang
Ctx = ctx:with_deadline_after(300, seconds),
Point = #{latitude => 409146138, longitude => -746188906},
{ok, Feature, HeadersAndTrailers} = routeguide_route_guide_client:get_feature(Ctx, Point).
```


CT Tests
---

To run the Common Test suite:

```
$ rebar3 ct
```

Interop Tests
---

The `interop` rebar3 profile builds with an implementation of the `test.proto` for grpc interop testing:


For testing grpcbox's server:

```
$ rebar3 as interop shell
```

With the shell running the tests can then be run from a script:

```
$ interop/run_server_tests.sh
```

The script by default uses the Go test client that can be installed with the following:

```
$ go get -u github.com/grpc/grpc-go/interop
$ go build -o $GOPATH/bin/go-grpc-interop-client github.com/grpc/grpc-go/interop/client
```

For testing the grpcbox client you can use the Go test server. But first, add `_ "google.golang.org/grpc/encoding/gzip"` to `server.go` imports or else the gzip tests will fail. Then simply build and run it:

```
$ go build -o $GOPATH/bin/go-grpc-interop-server github.com/grpc/grpc-go/interop/server
$ $GOPATH/bin/go-grpc-interop-server -port 8080
```

And run the interop client test suite:

```
rebar3 as interop ct
```

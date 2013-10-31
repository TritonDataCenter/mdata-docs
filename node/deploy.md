# Deploy

## <a name="domains"></a>Domains

TODO

## <a name="cluster"></a>Cluster

TODO

## <a name="hotload"></a>Reloading code at runtime

***TODO***

There are a few solutions for "hot loading" code out there, but they're
generally advised against as they often come with unintended consequences, such
as introducing memory/resource leaks that become difficult to track down, along
with other difficult to debug issues.

If you're running with a cluster setup, when you want to load new code you can
"close" the listener in the child, which then stops accepting new connections
(but the rest of the cluster continues on) once the child is finished with its
work you can let it die gracefully, then when the master spawns a new child in
its place it will then have the new code

## <a name="monitoring"></a>Monitoring

TODO

## <a name="testing"></a>Testing

TODO

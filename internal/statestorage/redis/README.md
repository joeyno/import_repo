# State storage
Open Match tries not to tie itself too closely to Redis, thus why the Redis code is largely sequestered off in this module.  However, it is expected that most MMFs will read and write directly from/to state storage without using Open Match as an intermediary, so enabling a different database as the internal state storage would likely require separate MMFs designed to work with that database.  We'll likely leave this to community members or software integrators, but we tried to avoid excluding the possibility for those who are sufficiently motivated. 

## Redis
The default state storage for Open Match is a _single instance_ of Redis.  Redis was chosen for the default for its proven performance and its alignment with a Matchmaker's resiliency expectations.  For many legacy matchmakers, a singleton approach means that the expected behavior upon loss of the matchmaker is that all state is gone and all clients should re-queue and all servers should be queried again for readiness.  Using Redis for Open Match adheres to the same assumptions under failure, but allows users to configure additional HA replicas for more resiliency and additional read replicas for horizontal scalability of performance.  Our evaluation is that Redis should be able to handle almost all game matchmaking scenarios with proper configuration. 

### Persistence
It is possible to [persist the Redis instance to disk](https://estl.tech/deploying-redis-with-persistence-on-google-kubernetes-engine-c1d60f70a043), but this is probably not a great solution for most matchmakers.  The performance hit for writing all the updates to disk with multiple tens or even hundreds of thousands of clients will likely make it untenable.  If you think you need persistence because you are worried about the Redis instance failing, you're probably better off with a HA Redis deployment as mentioned above.
### Resilience
Although it _is_ possible to go to production with this as the solution if you're willing to accept the potential downsides, for most deployments, a HA Redis configuration would better fit your needs.  An example YAML file for creating a [self-healing HA Redis deployment on Kubernetes](../install/yaml/01-redis-failover.yaml) is available.  Regardless of which configuation you use, it is probably a good idea to put some [resource requests](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/) in your Kubernetes resource definition as mentioned above.
### Single Redis instance considersations 
It is possible, as noted above, to run on a single Redis node if you know and can accept the consequences.  In general, if you're running something close to 'stock' Open Match with a single node Redis configuration, and the Redis instance is lost:
 * All players currently queued would need to re-queue.  They should get an error from the Frontend API if they are currently waiting for updates, but in case there is an edge case that's not covered, your client should be set up to retry after a reasonable amount of time without a response.
 * The Backend API clients would need to reconnect and request new matches.  The safest way would be to abandon all in-flight queries.
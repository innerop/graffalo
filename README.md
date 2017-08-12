
## GraphQL + Data-Oriented NodeJS Microservices Architecture with Realtime, "Stateless" UI 

A maintainable application architecture requires that the UI only contain the rendering logic and execute queries and mutations against the underlying data model on the server. A maintainable architecture requires that the UI be free of logic for deriving "app state" since that would embed business logic in the client. App state should be persisted to the database and then projected to the client via the mid tier, and refreshed as mutations occur on the server, which gives us a highly interactive, realtime UX that's free of incidental complexity under the hood.

With GraphQL we are able to define an easy-to-change application-level data schema on the server which captures the types and relationships in our domain model, and wiring it to data sources via resolvers that leverage a uniform data-oriented service API (CRUD methods plus Find with Query DSL that extends to the db or wraps existing API) to resolve client-specified "queries" and "mutations" against the domain model.

We use GraphQL to dynamically derive a UI-specific projection/shape of our app state from server-side data sources which is the updated in real time (with help of GraphQL Subscriptions)

With this approach, the developer's job becomes a much more pleasant and simpler task of building the application-level GraphQL schema, building UI components and declaratively specifying the required queries and mutations.

Client-centric architectures (like those based on client-side 'app state' stores like Redux) derive "app state" on the client, and as such end up embedding business logic in the client. While those architectures also enable things like 'optimistic UI' and 'offline first' apps they can exponentially complicate the work involved in maintaining and evolving the application.

![GraphQL](https://s29.postimg.org/b7ifw7vc7/image1.png)


## Some Challenges and Considerations

1. We should separate dynamic and static data, conceptually as well as physically. The two are often conflated and that creates architectural complexity. Static data is something like menu items in a food ordering app. Dynamic data is stuff like what the user has ordered and status of the order. While we may cache static data on the client, we should keep dynamic data on the server to avoid cache invalidation challenges. This principles followed here are intended for building realtime, transactional  apps (think: ordering & delivery and 'hard realtime' apps like Uber) In this context, we do not need to store dynamic data on the client because we're not deriving app state on the client (we do so in the GraphQQL resolvers in the mid tier) and this means that we don't reoplicate business logic on the client that belongs on the server. What we sacrifice here is the ability to update the UI optimistcally and the ability to work offline. But those two abilities are not needed for realtime, transactional apps. That is because it is pointless to tell the user that their order was successful or their Uber driver is 2 minutes away then find out from server that the order cannot be fulfilled or the driver is stuck in traffic. It just does not make any sense. This is to say that offline-first UI and optimistic UI are at odds with 'realtime' apps and present extra overhead of replicating business logic sand composing app state on the client, as well as storing dynamic data on the client that presents cache invalidation challenge. 

So then the question is wouldn't losing Optimistic UI hurt mobile performance? The fact is that GraphQL already does a lot to help mobile performance by allowing us to get exactly the data we need at any given instance as opposed to downloading more data than we need and putting it in the client side cache then having to keep all that data in sync with the server, which could be very challenging, especially if we derive app state on the client, which means that we have a lot of client derived state that depends on the cached dynamic data. A recipe for incidental complexity (cache invalidation challenge being one but also business logic creeping into client)
 

2. Avoid business logic in the client: “app state” should not be derived and/or cached on the client as that would put redundant business logic in the client which adds a lot of unneeded complexity to the app. The client needs to be just an I/O layer, fetching/re-fetching UI-component-bound query results and making mutations to data on the server. In this pure, functional client model, local component state (including local state of higher order components) is used for ephemeral, client-only state such as animation state. We avoid deriving app state on the client because that necessitates the embedding of business logic in the client, which complicates app building and maintenance. Instead, we opt to simply refetch UI-component-bound query results based on certain events like (backend mutation events, navigation state change or after network interruption) and let UI components derive their own local state from server-side app state which is persisted to the database and derived and projected to (or shaped for) the client in the mid-tier using GraphQL. The business logic stays on the server and the UI developer's job becomes very simple.

3. The need for pre-mutation hooks: we need to be able to authorize the user before executing a mutation (and conditionally avoiding the mutation) so we can implement authorization on the server and leave business logic completely outside the client.

4. The need for post-mutation hooks to make calls to external APIs (e.g. send email) after a given mutation. 

5. The need for live mutation events that perform well at scale: when something changes on the server, we need to know about it, either via polling or signaling. The latter has the advantage of being immediate.

6. Currently, GraphQL developers need to know many different querying syntaxes for the various resolvers in the schema (MySQL, MongoDB, different APIs, etc), or write their own API adapters for each data store to decouple their resolvers from the underlying data store.

7. We should be able to validate data based on our business logic, inside GraphQL resolvers

8. We should be able to implement secure authentication using OAuth and Email/Password.

9. We should be able to implement authorization independent of our database or network interface.

10. We should be able to implement validation generically at the UI component level for instant feedback as user types

11. We should be able to derive business logic in GraphQL resolvers using microservices with a data-oriented interface.

## Feathers: Data-Oriented Microservices Framework for NodeJS

The important new concept Feathers gives us in NodeJS is that of composable, data-oriented services. A service is just a JavaScript object that provides one or more of the following queries and mutations using the following data-oriented, uniform API (CRUD plus Find with Query DSL that extends to DB and wraps existing API), which is callable from any other service, and, therefore, composable. 

```javascript
// All service methods return a promise
var myService =
{
find: function(params) {},
get: function(id, params) {},
create: function(data, params) {}, // data can be an array (for multiple entry creation)
update: function(id, data, params) {},
patch: function(id, data, params) {}, // id may be omitted if one or more id's are provided in params (patch multiple entries)
remove: function(id, params) {}, // id may be omitted if one or more id's are provided in params (remove multiple entries)
setup: function(app, path) {}
}
// Use this service in your application at the /todos endpoint app.use('/todos', myService);
```

The key innovation in combining GraphQL with Features is about having composable, data-oriented services that have a uniform CRUD interface plus a Query DSL that is adapted for various backends (SQL, NoSQL, APIs, etc), where every table or collection in db is mapped to a service (as well as the possibility for each service to have its own database to remain decoupled) with GraphQL providing the application-level, expressive interface on top of that. This means that service composition replaces relational joins. For example, if we want to get all comments for a given post, we would not associate the posts table with the comments table (as that would violate the clean, maintainable pattern of having separate, decoupled services) and instead we would call the find(...) API on the comments service passing in the post id. We can also delete every comment that belongs to a given user using the same inter-service composition defined in the GraphQL resolvers, by calling the remove() API on the comments service passing in the user id.

Feathers will also hook into the mutative methods (create, update, patch, remove) and send events to listening clients when they successfully return. An application then can be made real-time just by listening to those events (with filtering rules on server side) and updating itself with the new data. It's important to note that backend services should not consume mutation events from Feathers services. The mutation events are meant for clients to update their state, not for other services to consume and opaquely mutate app state. If you want something to happen in an external system you should use Feathers after-hooks. If you want something to happen to Feathers connected data store after a given successful mutation then listen to the mutation event in the client that originated the mutation and create another mutation via GraphQL.

We use GraphQL schema to specify application-level data types and the relationships between them and we use GraphQL resolvers to specify which Feathers services to call in order to resolve the data type being requested (via Feathers services) and filter the result hierarchically before it's consumed by the client. Our GraphQL resolvers use the Feathers services to get the type being requested and all the types that it references. This means that the resolvers themselves are decoupled from the type of data store, which is a necessary for a clean, decoupled GraphQL implementation. In addition, 'before' and 'after' hooks in Feathers allow us to add things like authorization and sanitation (security layer) and any follow up action like send email to user. Validation can happen generically in the UI or as driven by business logic in the resolvers.   

## Data-Oriented Microservices

Let’s consider two entities, which each one of them is dealing with a different table in the database. The Deal Table is connected to Customer Table by Foreign Key (FK), so whenever we need information about the deals of a single customer, a simple SQL join will do the job.

![database](https://s29.postimg.org/55at5q6w7/image2.jpg)

In order to transform the whole relationship into a decoupled services approach we get rid of the FK. By doing this the service Deal will have to provide all the safeguards of an FK handling. In this particular case, every time a new request comes, we need to check if a deal does exist for the given customer and return only those deals.

Essentially, the services will be like this:

![services](https://s29.postimg.org/m4jrkzi3r/image3.jpg)

This radical approach brings up the following question:

With the old approach, a simple SQL join will bring us both the customer and deal information. Now, we need to perform two primary key lookups in order to get the same result (see Distributed Transactions and Concurrency Control.) Is the new approach slower because we need to make two calls to gather same information? The answer: it depends. We don't need SQL joins in this approach. Data is joined at the services layer and is normalized based on relations expressed in the GraphQL schema, which are implemented by resolvers which call into services, where each service encapsulates only one table/collection. There is no longer a relational join overhead. We're doing a simple lookup or update against a single table/collection. But the answer is that it depends on our schema design and the complexity of our query. 

However, the benefits in turning those entities into services are:
  
  1. The Deal and Customer tables can be in different databases (or schemas) without overwhelming a single database when we need to scale. Same for the Deal and Customer services.
  2. Maintenance of each service can be done easily, as each table doesn’t relate to anything else.

## Pagination

At this juncture, pagination of results can be done by either GraphQL or Feathers. My instinct is to use Windowing rather than pagination. This involves moving the fetch/display window over that data, or row at a time.

## Protecting Service Methods

There are some times where you may want to use a service method inside your application or allow other servers in your cluster access to a method, but you don't want to expose a service method publicly. We've created a bundled hook that makes this really easy.

```javascript
const hooks = require('feathers-hooks');

app.service('users').before({
  // Users can not be created by external access
  create: hooks.disable('external'),
});
```

## Managed Concurrent Mutations to Shared Resources

What happens in this kind of microservices architecture when we need to make multiple mutations (in promise chained calls) that are part of a single transaction (end state) across one or more services? Nothing if you have only one user and you're not interleaving writes. But what if you have two or more users concurrently using your application, with reads/writes against the same set of data?  The way this architecture is setup is we have inter-service composition happening in the GraphQL mutation resolvers. So for a transaction implementing dependent mutations asynchronously, the mutation resolver would orchestrate that transaction using conditional writes to guarantee consistency of our app state. If we're using a distributed database, for web scale transaction management, we would use one, like Google's Cloud Spanner, that can implement non-blocking concurrency control to guarantee consistency.

Some possible directions:

  1. Consistency Features of DynamoDB: https://quabase.sei.cmu.edu/mediawiki/index.php/Amazon_DynamoDB_Consistency_Features (also see DynamoDB Java transaction library)

  2. Consistency Features of Google's Cloud Spanner: https://cloud.google.com/spanner/docs/transactions

  3. Non-Blocking Optimistic Concurrency Control in SQL for Single DB Instance (see section below)

We should design the application to minimize write contention and for that we may use conditional updates to implement optimistic locking so that nothing is happening at the same time to the same resource or set of resources. We can also acquire read-write locks so nothing can happen to a resource between the time we check its state and the time we apply an dependent update to it or to another resource. That's pessimistic read-write locking. The onus is on us to design the app in a way that leads to minimal or no contention because contention will either slow things down or lead to inconsistent state. 

## Two-Phase Commit (2PC) 

While optimistic concurrency control (OCC) can take care of a whole class of problems, what happens if we're in the middle of a transaction that mutates (through promise-chained service invocations) many tables in our database and the application server crashes? The fact that we use data oriented microservices, where each service represents a table and that we may have dependent mutations that are carried out via separate services, means that we have to account for possibility of server crashing in the middle of a transaction. To solve this, many databases support a two-phase commit (2PC) process where we can rollback a transaction upon server failure or any logical failure. 

In a distributed DB scenario, Google Cloud Spanner also supports 2PC: https://cloud.google.com/spanner/docs/transactions and Amazon's DynamoDB supports this via a Java transaction library.

## Distributed Transactions

A distributed transaction includes one or more statements that, individually or as a group, update data on two or more distinct nodes of a distributed database. 

Databases that can be distributed may offer serializability or lineralizability or both (e.g. CockroachDB.)In a distributed database setting, the minimum required standard for transactional apps is serializability.  

## Optimistic Locking (for a single db instance)

The point is that Optimistic Locking is not a database feature, not for MySQL nor for others: optimistic locking is a practice that is applied using the DB with standard instructions.

Let's have a very simple example and say that you want to do this in a code that multiple users/clients can run concurrently.

This assumes use of Feathers-Knex rather than Sequelize and assumes one DB instance, not distributed.


SELECT data from a row having one ID field (iD) and two data fields (val1, val2)
optionally do your calculations with data
UPDATE data of that row

### The NO LOCKING way to is:

NOTE: all code {between curl brackets} is intended to be in the app code and not in the SQL side

```sql
-
SELECT iD,
       val1,
       val2
FROM theTable
WHERE iD = @theId;
 
- {code that calculates newVal1, newVal2} -
UPDATE theTable
SET val1 = @newVal1,
    val2 = @newVal2
WHERE iD = @theId;
 
- {GO ON WITH your other code}
```

### The OPTIMISTIC LOCKING way is:

NOTE: all code {between curl brackets} is intended to be in the app code and not in the SQL side

Note that the key point is in the structure of the UPDATE instruction and the subsequent number of AffectedRows check. It is these two things together that let your code realize that someone has already modified the data in between when you have executed the SELECT and UPDATE. 

```sql
-
SELECT iD,
       val1,
       val2,
       VERSION
FROM theTable
WHERE iD = @theId;
 
- {code that calculates newVal1, newVal2} -

UPDATE theTable
SET val1 = @newVal1,
    val2 = @newVal2,
    VERSION = VERSION + 1
WHERE iD = @theId
  AND VERSION = @oldversion;
 
- {IF AffectedRows == 1 } 
- {proceed with other code} 
- {ELSE} 
- {decide what to do since it has gone bad} 
- {endif}
```

Here is shown that we can use a dedicated field (that is modified each time we do an UPDATE) to see if anyone was quicker than us and changed the row between our SELECT and UPDATE. 

## Client State

When we say that we must "derive" client state from server-side app state we mean that we must not be derive app state in the client. We should derive state of the client in the mid tier using persisted app state from database or API and deriving and then shaping/projecting app state to client via GraphQL. This way we don't end up with domain specific logic in the client. This means that every value in the client is derived directly from the server-side app state. 

The Apollo GraphQL client, which we've chosen to use in this architecture, persists query results in a local cache (for the purpose of caching and/or deriving app state in the client for Optimistic and Offline-First UI implementations) The use of forceFetch (bypass cache) in Apollo client can work nicely with Feathers realtime mutation events in that we may set forceFetch each time we receive a mutation event relevant to the route (in addition to when landing on a given route.)

After loading the data, instead of having to fetch app state from the server continuously to keep client state in sync, which does not scale, Feathers realtime events save the day by telling us only when there has been a change to the data. This means that each route on the client has to listen to those events that relate to it so it can update its data, and when the route comes up again we would fetch again from the server. 

We leave it to the client to decide exactly what part of that area of app state it wants to refresh. So if you have e.g. a grid of 100K rows and UI is displaying rows 20-60, we won't care if rows 500-2500 have changed, so we don't need to push that data to the client. But we do have to tell the client which rows changed. 

The local state of the component can be used for query params such as the index of the last fetched item in a social status feed, so the status feed component, upon receiving a mutation signal from the server that is relevant to it, can get the latest items (from last index, which is passsed in as query param) We may generalize this pattern to UI components with arbitrary data structures. We do this because there is no way for the server to know what was the last state of the status feed on the client without the server becoming stateful and losing horizontal scalability or reliability.

For complex components like forms, a good pattern is for local component state to be lifted into an auto-generated higher order component (HOC) wrapper so that all client components will be functionally pure and local state management will be in the HOC, clearly separated from the implementation.

So while app state is not derived on the client things are different when it comes to client state:

1. For client components that issue GraphQL mutations, we should keep transient state of the resource(s) being mutated in a local route-centric store, where each route has its store that can be passed into the next route upon route transition to seed initial state from current route, until the transient state is committed to the server. A good example is a multi-step food ordering UI, where you select your pizza crust then the toppings then the sides etc. The idea is that we should be able to go back and pick a burger instead if a pizza and not end up with an inconsistent state having a burger with peperoni for topping. 

2. Each route has a main container component associated with it and it's that component that does initial route hydration (and keeping in sync with server via server mutation events) but each sub-component of the route can launch it's own mutations and other sub-components in the route will just react to server mutation events if their data did change due to a mutation launched by that sub-component, or due to a mutation from another user involving a shared resource that the sub-component depends on for its state. This means that all components have to state their dependencies (statically) and the sync-ing happens with server mutation events.   

3. For client-only state (i.e. any state that is not part of 'app state', e.g animation and client side validation) we should keep that state in the local state of the HOC wrapper.   


## Horizontal & Vertical Scaling 

At this point, Feathers team has yet to document vertical scaling (multi-core support via Node cluster) and how that works with sockets.

Feathers can be scaled horizontally with Nginx handling the load balancing. Feathers can be Dockerized with one process per container and then scaled via Kubernetes cluster or in a more automated fashion with AWS Elastic Beanstalk and AWS Elastic Load Balancer.

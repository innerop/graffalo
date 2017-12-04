
## Making 'Change' a First-Class Citizen in Platform Architecture

![GraphQL](https://image.ibb.co/bDaxcw/Untitled_Diagram_34.png)

## Work In Progress

### GraphQL: From Anti-Patterns To Effective Patterns

Prior to adopting GraphQL, we observed that UI developers, using modern frameworks like React and Angular 4, will often isolate and model app state in the UI in denormalized form 

Example: both objects A and B references object C, so two copies exist of object C 

This means that every time we find out that C changed we need to update it in two (or many) places, which leads to messy state management and increased chance for communicating inconsistent/invalid state to the user and/or server

One solution advertised by Redux author is his Normalizr library which normalizes/denomralizes data in one synchronous call and degraded UI performance by blocking longer than the frame budget when used on the client when the data structures are large (when fetching data from the API) and/or have deeply nested relations once normalized (when populating the UI with data)  

Moreover, if the API output results in many repeated entries (e.g. getting movie times for the day across many theaters) the JSON to be parsed will be larger than it needs to be if the API output was de-duped (with type name and id's remaining in output, but not the content of each item)

The best solution we found to the above is to use GraphQL ands start with a normalized data model in the GraphQL Schema with CRUD microservices corresponding to the user-defined data types then de-normalize (with de-duping using the graphql-deduplicator library) using async resolvers on the server, on-demand, according to the shape of the UI component to be rendered, then cache static data (only) in the UI and, use the Apollo GraphQL Client, to denormalize fully from the cache (or refresh cache first from server for dynamic data, e.g. shopping cart) and render UI.  

Another anti-pattern UI developers often implement is fetching object A, object B and (conditionally) object C from the server then composing object D in the client, possibly with some new fields derived on the client side. This leads to three major problems, the first problem being the data aggregation logic is imperatively coded in the UI and as requirements change there is a lot of rework, and the second and third problems being the leaking of business logic into the UI as service orchestration logic and data derivation logic.

With GraphQL and CRUD Microservices, we get to aggregate data declaratively by simply defining the data types, including derived fields, and the relations between the types in GraphQL schema and coding the query/mutation resolvers (orchestration) and the data type resolvers (aggregation, derivation)    

This way business logic remains out of the UI and the UI state tree can be composed as a pure projection of app state on the server. 

### The Future: From Resuable Components To Resuable Custom Elements

There are two fundamental W3C-specified Web Components technologies built and polyfill'ed (with intent to build) into modern browsers: 

Shadow DOM (for DOM/CSS encapsulation) and 
Custom Elements (user-defined native browser components with state and lifecycle hooks)

SkateJS allows us to export components rendered by React, Angular 1.5, 2, 4, Vue, and other modern composition-oriented frameworks as W3C Custom Elements and and compose those Custom Elements into higher order Custom Elements with data flowing from parent to descendants and so on. 

With SkateJS we can have an Angular 4-rendered Form custom element composing Angular 2-rendered input and selection custom elements, or the other way around with Angular 2-rendered Form custom element composing Angular 5 --and Angular 4-- rendered input and selection custom elements.

SkateJS also provides a Router that is framework agnostic. Only the rendering part of any framework (Angular, React, Preact, Vue, et al) is ever used.  

This way we can migrate from Angular 4 to Angular 5 or any other composition-oriented framework in piecemeal fashion, as opposed to having to do it all at once, which is almost always an unrealistic approach, since our components are essentially all custom elements regardless of rendering library (different versions of Angular, React, Preact, Vue, et al)


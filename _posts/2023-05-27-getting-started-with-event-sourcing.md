---
tags:
  - Event Sourcing
---

# Getting started with Event Sourcing
For 5 years I've been working with Event Sourcing, and in my 10 years as a developer I've heard these phrases a lot:

> I wish I knew how we got into this state

> Who changed this bit of information?

> Can we please do a change or audit log on this and that?

Even though I really like Event Sourcing for persisting data, it comes with some (maybe a lot of) complexity, and it is often not easy for new employees to get into.  
Event Sourcing is not easy - the same goes for SQL, we have just been taught how to do it a looong time ago. If we (for now) ignore some of the more complex parts - like projections, processes, snapshots, versioning etc - it isn't that difficult and can be seen as Document Store.


## What is Event Sourcing?
Event Sourcing is a way of persisting your data. Instead of storing state (like `User("Andreas")`), you store the events leading to that state (`UserCreated("Andreass"), NameUpdated("Andreas"`). To get the current state you simply apply the relevant events.

### So is it ...?
No it is not more than this.
- It is not a way for to communicate between Micro Services.
- It is not CQRS or DDD - but can be used for it and is a very good fit!
- It have nothing to do with DOM events.

It is just one way of storing data. 


## Let's get started
To get started with Event Sourcing you need you favorite programming language (I will use TypeScript) and an Event Store (I suggest using [EventStoreDB](https://www.eventstore.com/) or implementing a simple Event Store in memory for playing a bit around). In the following it will be used (more or less) as a Document Store - we have documents, but have to keep track of events instead of state.


## Components
There are three components:
- EventStore - _Keeps track of events_
- Repository - _Loads and saves documents_
- Document - _An abstract class for building your documents_


### The Event Store
The event store will have the following methods
- _Read all events from a stream (events belonging to a given document):_  
  `read(streamId: string): Promise<ReadonlyArray<object>>`
-  _Append events to a stream (again belonging to a document):_  
  `append(streamId: string, expectedEventCount: number, events: ReadonlyArray<object>): Promise<void>`  
  _The expectedEventCount is for optimistic concurrency_

With EventStoreDB it is fairly trivial to implement an adapter between our simple interface and the more complex interface of EventStoreDB. Basically it is just two calls and some mapping

```typescript
  const events = client.readStream(documentId);
  // map events to 
```

and

```typescript
  client.appendToStream(
    documentId,
    events, // map it to event data
    {
      expectedRevision: expectedEventCount // map it to AppendExpectedRevision
    }
  )
```


### The document
The document helper class can be implemented in various ways. The important thing is to have a way to emit events and a way to apply them.  
Let's start by designing the api. I've often seen something like this:

```typescript
class User extends Document {
  // State
  #name: string;
  get name() {
    return this.#name;
  }
  set name(name: string) {
    if (name === this.#name) return;
    // nothing is changed here - it just emits the event
    this.emit(new NameUpdated(name))
  }

  // State management
  apply(event: object) {
    if (event instanceof UserCreated) {
      this.#name = event.name;
    }
    if (event instanceof NameUpdated) {
      this.#name = event.name;
    }
  }

  static createUser(name: string) {
    const user = new User();
    user.emit(new UserCreated(name));
    return user;
  }
}
```

In my opinion the apply method gets quite bloated, so I prefer are more functional style for this:

```typescript
class User extends Document {
 // State
  #name: string;
  get name() {
    return this.#name;
  }
  set name(name: string) {
    if (name === this.#name) return;
    this.emit(new NameUpdated(name))
  }

  // State management
  constructor() {
    super();
    this
      .on(UserCreated, e => this.#name = e.name)
      .on(NameUpdated, e => this.#name = e.name)
    ;
  }

  static createUser(name: string) {
    const user = new User();
    user.emit(new UserCreated(name));
    return user;
  }
}
```

Beside from this api, the repository needs to keep track of events, so a generic onEvent method is needed as well. A simple implementation of this would be:

```typescript
abstract class Document {
  #listeners = [] as Array<(e: object) => void>

  emit(e: object) {
    for (const listener of this.#listeners) {
      listener(e);
    }
  }

  on<T>(TEvent: new(...args: any) => TEvent, handler: (e: TEvent) => void) {
    this.#onEvent(e => {
      if (e instanceof TEvent) {
        handler(e);
      }
    });
  }

  onEvent(handler: (e: object) => void) {
    this.#listeners.push(handler);
  }
}
```

### The repository
Now we just need a repository with a load and a save method. It just reads events from the event store, applies them to the document and tracks the events afterwards:

```typescript
class Repository {
  readonly #trackedDocuments = new Map<Document, {
    id: string
    trackedEvents: object[]
    readonly expectedEventCount: number
  }>();

  readonly #eventStore: EventStore;
  
  constructor(
    eventStore: EventStore
  ) {
    this.#eventStore = eventStore;
  }

  async load<TDocument extends Document>(
    Document: new () => TDocument,
    id: string
  ): Promise<TDocument> {
    const doc = new Document();

    const events = await this.#eventStore.read(id)
    const trackedEvents = [] as object[];
    for (const event of events) {
      // apply all existing events to restore state
      doc.emit(event);
    }

    // track future events
    doc.onEvent(e => trackedEvents.push(e));

    this.#trackedDocuments.set(doc, {
      id,
      trackedEvents,
      expectedEventCount: events.length
    });

    return doc;
  }

  async save(document: Document) {
    const { id, trackedEvents, expectedEventCount } = this.#trackedDocuments.get(document) ?? {};
    await this.#eventStore.append(id, expectedEventCount, trackedEvents);
    this.#trackedDocuments.delete(document);
  }
}
```

## Putting it all together
Now these building blocks can be used as a regular document store in your services:

```typescript
class MyService {
  constructor(
    private readonly repository: Repository
  ) {}

  async updateUserName(id: string, name: string) {
    const user = await this.repository.load(User, id);
    user.updateName(name);
    await this.repository.save(user);
  }

  async getUser(id: string) {
    return this.repository.load(User, id);
  }
}
```

## A few thoughts
So this is a way to get started with event sourcing, but you should probably combine it with Domain Driven Design (DDD) and CQRS.
- For DDD, the document becomes the _AggregateRoot_. Everything within a stream is consistent. Further your events most likely becomes your Domain Events or even Integration Events, and you will add all your business logic inside the _AggregateRoot_.  
With the given/when/then test pattern, it becomes incredible easy to test your domain - think of it as: __Given__ _some event_, __when__ _some action_ __then__ _some events_ should be appended to the stream.
- When it comes to CQRS, you should use your _Document_ as the write side. Whenever an event occurs, you should have a mecanism that reads the event and writes whatever change you need to read model. You can use whatever storage you want for you read models, and make very efficient reads.
- When serializing events, the prototype is lost, so you should either explicitly set the prototype when deserializing you events, or add a unique property to your events so you can use that instead of `instanceof` when applying events.

## Conclusion
With these simple building blocks, it is fairly easy to get started with Event Sourcing. The biggest hurdle is how/when to use emit and when to use `.on` (or `apply`). Much of the benefit of Event Sourcing have been ignored here, but this is a start and you application can now adapt to a full blown Event Sourced system.
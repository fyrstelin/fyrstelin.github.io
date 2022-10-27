---
tags:
  - Event Sourcing
---

# Getting started with Event Sourcing
For 5 years I've been working with Event Sourcing. Even though I really like that way of persisting data, it comes with some (maybe a lot of) complexity, and it is often not easy for new employees to get into.  
Event Sourcing is not easy - the same goes for SQL, we have just been taught how to do it a looong time ago. If we (for now) ignore some of the more complex parts - like projetions, processes, snapshots, CQRS etc - it isn't that difficult and can be seen as Document Store.


## What is Event Sourcing?
Event Sourcing is a way of persisting you entities. Instead of storing the state (like `User("Andreas")`), you store the events leading to that state (`UserCreated("Andreass"), UserNameUpdated("Andreas"`). To get the state you simply apply all events for the given entity.  

### So is it ...?
No it is not more than this.
- It is not a way for to communicate between Micro Services.
- It have nothing to do with DOM events.
- It is not CQRS or DDD - but can be used for it and is a very good fit!

It is just one way of storing data. 


## Let's get started
To get started with Event Sourcing you need you favorite programming language (I will use TypeScript) and an Event Store (I suggest using [EventStoreDB](https://www.eventstore.com/) or implementing a simple Event Store in memory for playing a bit around). In the following it will be used (more or less) as a Document Store - we have documents, but have to keep track of events instead of state.


## Components
There are three components:
- EventStore - _Keeps track of events_
- Repository - _Loads and saves documents_
- Document - _An abstract class for building (and testing) you documents_


### The Event Store
The event store will have the following methods
- _Read all the events belonging to a given document:_  
  `read(documentId: string): Promise<IReadonlyList<object>>`
-  _Append events to a given document:_  
  `append(documentId: string, expectedEventCount: number, events: IReadonlyList<object>): Promise<void>`  
  _The expectedEventCount is to ensure that nothing else have happened since reading the events (optimistic concurrency)_

With EventStoreDB it is fairly trivial to implement an adapter between our simple interface and the more complex interface of EventStoreDB. Basically it is just a call to

```typescript
  const events = client.readStream(documentId);
  for await (const event of events) {
    // collect into list
  }
```

and

```typescript
  appendToStream(
    documentId,
    events, // map it to event data
    {
      expectedRevision: expectedEventCount // map it to AppendExpectedRevision
    }
  )
```


### The document
The document helper class can be implemented in various ways. The important thing is to have a way to emit events and a way to apply them. You should emit events when your business logic allows it, and apply them to build your state.  
Let's start by designing the api. I've often seen something like this:

```typescript
class User extends Document {
  // State
  #name: string;
  get name() {
    return this.#name;
  }
  private set name(value: string) {
    this.#name = value
  }

  // State management
  apply(event: object) {
    if (event instanceof UserCreated) {
      this.name = event.name;
    }
    if (event instanceof UserNameUpdated) {
      this.name = event.name;
    }
  }

  // Business logic
  createUser(name: string) {
    if (!name) throw new Error('Please provide name')
    this.emit(new UserCreated(name));
  }

  updateName(name: string) {
    if (name !== this.name) {
      this.emit(new UserNameUpdated(name));
    }
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
  private set name(value: string) {
    this.#name = value
  }

  // State management
  constructor() {
    super();
    on(UserCreated, e => this.name = e.name);
    on(UserNameUpdated, e => this.name = e.name);
  }

  // Business logic
  createUser(name: string) {
    if (!name) throw new Error('Please provide name');
    this.emit(new UserCreated(name));
  }

  updateName(name: string) {
    if (name !== this.name) {
      this.emit(new UserNameUpdated(name));
    }
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
    this.#listeners.push(e => {
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
  #trackedDocuments = new Map<Document, {
    id: string
    trackedEvents: object[]
    readonly expectedEventCount: number
  }>();
  
  constructor(
    private readonly eventStore: EventStore
  ) {}

  async load<TDocument extends Document>(
    Document: new () => TDocument,
    id: string
  ): Promise<TDocument> {
    const doc = new Document();

    const events = await this.eventStore.read(id)
    const trackedEvents = [] as object[];
    for (const event of events) {
      doc.emit(event); // all these events have happened, so no need to validate any business logic
    }

    // track future events
    doc.onEvent(e => trackedEvents.push(e));

    this.#trackedDocuments.put(doc, {
      id
      trackedEvents,
      expectedEventCount: events.length
    });

    return doc;
  }

  async save(document: Document) {
    const { id, trackedEvents, expectedEventCount } = this.#trackedDocuments.get(document);
    await this.eventStore.append(id, expectedEventCount, trackedEvents);
    this.#trackedDocuments.delete(document);
  }
}
```

## Using it all together
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

## Testing
As a short note, your business logic is fairly easy to test following the Arrange-Act-Assert (or Given-When-Then) pattern:

```typescript
it('should update user name', () => {
  // Arrange / Given
  const user = new User();
  user.emit(new UserCreated('Andreas Kristiansen'));

  const trackedEvents = [] as object[]
  user.onEvent(e => trackedEvents.push(e));

  // Act / When
  user.UpdateName('Andreas Fuller');

  // Assert / Then
  expect(trackedEvents).toEqual([
    new UserNameUpdated('Andreas Fuller')
  ])
});
```

More on tests another day...

## Some considerations
- When serializing events, the prototype is lost. The name is stored in EventStoreDB, so you should either explicitly set the prototype when deserializing you events, or add a unique property to your events so you can use that instead of `instanceof` whn applying events.
- You probably want an interface or abstract class for events instead of using object.

## Conclusion
With these simple building blocks, it is fairly easy to get started with Event Sourcing. The biggest hurdle is implementing the business logic in the document classes and how/when to use emit and when to use `.on` (or `apply`). Much of the benefit of Event Sourcing have been ignored here, but this is a start and you application is now extendable with projections. Further your business logic (probably the most essential part of your code base) is very easy to test.
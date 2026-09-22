---
tags:
  - .NET
---

# Union Types in .NET 11
## The most hyped and disappointing feature

For the last few years I've worked a lot with a Result&lt;T&gt; abstraction in .NET, and for the most part it worked quite well. But I was really looking forward to getting unions in .NET. The proposal looked promising, but the result is really disappointing; at best, it is a step towards something useful.

## A brief introduction

Unions in .NET 11 are pretty straightforward. They are a closed set of types and are defined like this:

```c#
public class Response;
public class Error;

public union Result(Response, Error);
```

Now you can switch on the union:

```c#
Result result = service.DoSomething();

switch (result) {
    case Response res:
        // Do something
        break;
    case Error err:
        // Do something else
        break;
}
```

It looks like another way of creating type hierarchies—and it is, just the other way around. The relation is defined on the super type, and the subtype does not know anything about the super type.

## It brings nothing new

I am not really seeing anything new here. This is what OneOf and Either have been doing for years, and it solves everything more or less in the same lines of code.

```c#
// Yeah okay - this is a stretch. Inherit OneOfBase and add implicit conversions
global using Result = OneOf.OneOf<Response, Error>;

public class Response;
public class Error;

Result result = service.DoSomething();

result.Switch(
    (Response res) => {}, // Do something,
    (Error err) => {} // Do something else
); 
```

## What I am missing

I want the possibility to merge unions. In my pipelines I am often composing services in a way that produces different errors. I can express that union, but it just isn't good enough:

```c#
var result = request
    .Validate() 
    // ^ Result<ValidationError>
    .Then(() => ctx.Users.Get(request.UserId))
    //                    ^ Result<User, NotFoundError>
    .Convert(user => user.UpdateEmail(request.Email))
    //                    ^ Result<ValidationError>
    .Convert(_ => ctx.SaveChanges());
    //              ^ Result<ConcurrencyError>

return result;
  //   ^ Result<Either<ValidationError, Either<NotFoundError, Either<ValidationError, ConcurrencyError>>>>
```

This nested Either is not readable, and it does not treat the duplicate ValidationError as the same. It gets worse when you try to handle it:

```c#
switch (result) {
    case Success:
    case Either<ValidationError, Either<NotFoundError, Either<ValidationError, ConcurrencyError>>> error:
        switch(error) {
            case ValidationError:
            case Either<NotFoundError, Either<ValidationError, ConcurrencyError>> innerError:
            switch (innerError) {
                case NotFoundError:
                case Either<ValidationError, ConcurrencyError>:
                // You get the point
            }
      }
}
```

I could extend the Either abstraction to multiple type arguments, but that makes the methods for the Result abstraction ambiguous, and still it cannot remove duplicate types.

So what I really want is something that didn't make it into .NET 11: the [ad hoc unions](https://github.com/dotnet/csharplang/blob/main/meetings/working-groups/discriminated-unions/TypeUnions.md?utm_source=chatgpt.com#ad-hoc---ad-hoc-unions).

With that, the same example would be:

```c#
var result = request
    .Validate() 
    // ^ Result<ValidationError>
    .Then(() => ctx.Users.Get(request.UserId))
    //                    ^ Result<User, NotFoundError>
    .Convert(user => user.UpdateEmail(request.Email))
    //                    ^ Result<ValidationError>
    .Convert(_ => ctx.SaveChanges());
    //              ^ Result<ConcurrencyError>

  // result
  // ^ Result<ValidationError or NotFoundError or ConcurrencyError>

switch (result) {
    case Success:
    case ValidationError:
    case NotFoundError:
    case ConcurrencyError:
}
```

I am so disappointed that ad hoc unions were left out. They would have made a difference, whereas the unions we got are just an alternative to OneOf and the likes.

## devblogs.microsoft.com
[Link](https://devblogs.microsoft.com/dotnet/csharp-15-union-types/)

This is a classic toy example:

```c#
public record class Cat(string Name);
public record class Dog(string Name);
public record class Bird(string Name);

public union Pet(Cat, Dog, Bird);

Pet pet = new Dog("Rex");

string name = pet switch
{
    Dog d => d.Name,
    Cat c => c.Name,
    Bird b => b.Name,
};
```

If that were rewritten using OneOf, it would require a bit more code for the Pet abstraction. But for callers it would actually be a bit less. So I don't think it really makes a difference.

```c#
public record class Cat(string Name);
public record class Dog(string Name);
public record class Bird(string Name);

public class Pet(OneOf<Cat, Dog, Bird> input) : OneOfBase<Cat, Dog, Bird>(input)
{
    public static implicit operator Pet(Cat cat) => new(cat);
    public static implicit operator Pet(Dog dog) => new(dog);
    public static implicit operator Pet(Bird bird) => new(bird);
}

Pet pet = new Dog("Rex");

string name = pet.Match(
    dog => dog.Name,
    cat => cat.Name,
    bird => bird.Name
);
```


## Final thought

Even though I am very sceptical of the current Release Candidate for Unions, I am optimistic for the future. This is a step towards something useful (ad hoc unions). I am a bit afraid that we won't get there though, but I keep my fingers crossed. Until then, I will probably stick with OneOf or similar.
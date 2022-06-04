# sBackend Architecture

Our microservices are using .NET 6 and docker. Each microservice can be instantiated using our [template](), which provides a base for anything we might need, including tests. The template is composed by 5 different projects:

- `src/MamisSolidarias.WebApi.TEMPLATE`: This is the main Web API project, where the endpoints will be created.
- `test/MamisSolidarias.WebApi.TEMPLATE`: This project has all the tests the Web API project.
- `src/MamisSolidarias.HttpClient.TEMPLATE`: This is an Http Client implemented by us to simplify communication with other microservices.
- `test/MamisSolidarias.HttpClient.TEMPLATE`: This project contains all the tests for the Http Client project.
- `src/MamisSolidarias.Infraestructure.TEMPLATE`: This project contains all references to the DB.

## Web API

The Web API project is the microservice that we will deploy on our servers as a docker container. This project will follow the [REPR](https://deviq.com/design-patterns/repr-design-pattern) pattern. To put it simply, this design pattern states that all endpoints need to have 3 basic pieces:

- The Endpoint
- A Request
- A Response

The idea is to ditch controllers as they get way too big and complex, and focus on what each endpoint does. To support this, we will use the library [FastEndpoints](https://fast-endpoints.com/). This library implements this pattern as well as security and validation out of the box while performing better than traditional controllers.

Our file directory should look something like this:

```
Endpoints
└── Foo
    ├── Endpoint.cs
    ├── Request.cs
    └── Response.cs
```

Where each file contains:

- `Endpoint.cs` will contain all the logic for the actions of the endpoint. This includes communication with the database and other microservices.
- `Request.cs` will contain the Request model as well as the validation logic for it.
- `Response.cs` will contain only the Response model.

All of these classes must be **internal** except for the Request and Response models, as these will be used for the HTTP Client.

### Fast Endpoints

Fast endpoint provides a lot of APIs we can play with, but on this document we will focus on the basic things we will use on most (if not all) the endpoints.

Our endpoints should look something like this:

```csharp
internal class Endpoint: Endpoint<Request,Response>
{
    public override void Configure()
    {
        Get("/foo");
    }
    
    public override async Task HandleAsync(Request req, CancellationToken ct)
    {
      // Do something...
      await SendOk(ct);
    }
}
```

The `Configure` function will set up our endpoint and how it can be called. We can call any of the following functions:

- `Get/Put/Post/... (string path)`: This is the only **mandatory** function we have. It will define the route of our endpoint as well as the method used to access it.
- `AllowAnnonymus()`: It says that the endpoint can be called without being authenticated.
- `Policies(string policy)`: It states that only users with the correct security policy can access this endpoint

There are many more functions that we can call here, and if needed they can be found in the documentation for FastEndpoints.

The `HandleAsync` function will be called to resolve our request. The request model should come preloaded with all the required information and we can simply implement our logic here. To return a specific status code, we can use the following functions:

- `SendOk()`: It will send a 200 OK response.
- `SendNotFound()`: It will send a 404 NOT FOUND response.
- `SendCreated()`: It will send a 201 CREATED response.
- …

Also, if we want to send something in our response body, we can use the function `SendAsync`. This function accepts both an object and a status code (it will default to 200).

#### Model Binding

Models are automatically mapped from their HTTP Request into our class.  While FastEndpoints has its own logic to understand if the properties in a class are coming from the request body or the headers for example, we will not rely on it, as it makes the code less legible.

This library provides annotations that we can use on our properties to not only tell them where it should look for the parameters, but it also allows us to tell it the name it has on the HTTP Request. We can see an example here:

```csharp
public class Request
{
  [FromRoute]
  public string Name { get; set; }
  
  [FromHeader(Name = "id")]
  public int Id { get; set; }
  
  [FromBody(Name = "address")]
  public Address HomeAddress { get; set; }
}
```

We can use any of the following annotations:

- `FromRoute`
- `FromBody`
- `FromClaim`
- `FromForm`
- `FromHeader`
- `FromQuery`

To learn more about model binding in FastEndpoints, check the [docs](https://fast-endpoints.com/wiki/Model-Binding.html).

#### Validation

Fast Endpoints implements **model validation** using FluentValidation. This is another important library we are using as it allows us to do validation very quickly and if there is an error it will automatically emit an error 403 HTTP response with the invalid fields. In the following example we will implement a validation for the model defined in the model binding example:

```csharp
public class RequestValidator : Validator<Request>
{
    public RequestValidator()
    {
        RuleFor(t => t.Name)
            .MinimumLength(3).WithMessage("Names cannot be shorter than 3 characters")
            .MaximumLength(50).WithMessage("Names cannot be longer than 50 characters");

        RuleFor(t => t.Id)
            .NotNull().WithMessage("the Id is mandatory");
        
        RuleFor(t=> t.HomeAddress.Street)
            .NotNull().WithMessage("The home address must have a street name")
            .MinimumLength(3).WithMessage("Street names cannot be shorter than 3 characters")
            .MaximumLength(100).WithMessage("Street names cannot be longer than 50 characters");

        RuleFor(t => t.HomeAddress.HouseNumber)
            .GreaterThan(0).WithMessage("Street numbers cannot be negative");
    }
}
```

FluentValidation provides a million different ways to validate our models, including localization, conditions, complex models and inheritance. To learn more about this library, check out the [docs](https://docs.fluentvalidation.net/en/latest/index.html).

But validation does not stop at the model. We can actually use this to **validate our application logic**. Fast Endpoints provides the following functions to check for errors in our code:

- `AddError(string err)` adds a validation failure to the response, but it does not stop the execution of it.
- `ThrowIfAnyErrors()` will stop execution and return an error response if any errors where added prior to the execution of this function.
- `ThrowError(string error)` will stop the execution and return an error response with all the previous errors detected, as well as the current error detected.

To learn more about validation, check out the [docs](https://fast-endpoints.com/wiki/Validation.html#application-logic-validation).

#### Dependency Injection

Even though Fast Endpoints supports 3 different ways to get services injected using dependency injection, we will only use proper injection. For this, we have to create a **public** property of the desired service’s interface and the library will take care of the rest:

```csharp
public class MyEndpoint : EndpointWithoutRequest
{
    public IHelloWorldService HelloService { get; set; }

    public override void Configure()
    {
        Get("/api/hello-world");
    }

    public override async Task HandleAsync(CancellationToken ct)
    {
        await SendAsync(HelloService.SayHello());
    }
}
```

Also, our endpoints come preloaded with basic services such as:

- `ILogger`: the logger service comes preloaded on the `Logger` property.
- `IConfiguration`: We can access all of our configuration form the `Config` property.
- `IWebHostEnvorinment`: We can access al of our environment variables from the `Env` property.

### Docker

Each microservice comes with a preconfigured file called `DOCKERFILE`. This file has all the instructions to automatically compile and create a docker image and nothing it should not be modified unless necessary. In this article we will explain what the current DOCKERFILE does and why we use it. Here’s the complete file:

```dockerfile
FROM --platform=linux/amd64 mcr.microsoft.com/dotnet/runtime-deps:6.0-alpine  AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:6.0-alpine AS publish
WORKDIR /src
COPY ["MamisSolidarias.WebAPI.TEMPLATE/MamisSolidarias.WebAPI.TEMPLATE.csproj", "MamisSolidarias.WebAPI.TEMPLATE/"]
RUN dotnet restore "MamisSolidarias.WebAPI.TEMPLATE/MamisSolidarias.WebAPI.TEMPLATE.csproj"
COPY . .
WORKDIR "/src/MamisSolidarias.WebAPI.TEMPLATE"
RUN dotnet publish "MamisSolidarias.WebAPI.TEMPLATE.csproj" -c Release -o /app/publish \
                                                                   --runtime alpine-x64 \
                                                                   --self-contained true \
                                                                   /p:PublishTrimmed=true \
                                                                   /p:PublishSingleFile=true 

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["./MamisSolidarias.WebAPI.TEMPLATE"]

```

We can see that this file is split into 3 main parts, delimited by the keyword `FROM`.

In the first section we are only creating an image and exposing the http (80) and https (443) ports. The base image is `dotnet/runtime-deps:6.0-alpine`, we chose this image because it contains only the absolute necessary dependencies for NET6 and it is super light-weight compared to the rest of the official dotnet images.

As for the platform parameter, some people develop using ARM machines and docker would create an ARM image, that is not compatible with the current server less offerings of Azure. With this parameter we are forcing docker to always use the amd64 version.

```dockerfile
FROM --platform=linux/amd64 mcr.microsoft.com/dotnet/runtime-deps:6.0-alpine  AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443
```

On the second part of our file we are building and generating our binaries. We first restore our dependencies and then we will compile it using the flags `self-contained` and `publishTrimmed`. This means that our docker image will not need to have the NET6 runtime installed, the app will contain the whole runtime. This will make our image quite big, so `publishTrimmed` will remove all the code from the runtime that I am not using on my application.

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:6.0-alpine AS publish
WORKDIR /src
COPY ["MamisSolidarias.WebAPI.TEMPLATE/MamisSolidarias.WebAPI.TEMPLATE.csproj", "MamisSolidarias.WebAPI.TEMPLATE/"]
RUN dotnet restore "MamisSolidarias.WebAPI.TEMPLATE/MamisSolidarias.WebAPI.TEMPLATE.csproj"
COPY . .
WORKDIR "/src/MamisSolidarias.WebAPI.TEMPLATE"
RUN dotnet publish "MamisSolidarias.WebAPI.TEMPLATE.csproj" -c Release -o /app/publish \
                                                                   --runtime alpine-x64 \
                                                                   --self-contained true \
                                                                   /p:PublishTrimmed=true \
                                                                   /p:PublishSingleFile=true 
```

On the last section, I am copying the resulting binaries into the final image and running the entry point.

```dockerfile
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["./MamisSolidarias.WebAPI.TEMPLATE"]
```

### Security

Security is build in ASP.NET Core. To enable authorization and authentication we have to register the respective middleware in the file `StartUp/MiddlewareRegistrator.cs`:

```csharp
internal static class MiddlewareRegistrator
{
    public static void Register(WebApplication app)
    {
      	...
        app.UseAuthentication();
        app.UseAuthorization();
      	...
    }
}
```

Then we need to configure our authentication policies on the file `StartUp/ServiceRegistrator.cs`:

```csharp
internal static class ServiceRegistrator
{
    public static void Register(WebApplicationBuilder builder)
    {
      	...
        builder.Services.AddAuthorization(
            options => options
                .AddPolicy(
                    "admin", 
                    policy => policy.RequireAuthenticatedUser().RequireRole("admin")
        ));
      	...
    }
}
```

This will register a policy with the name `admin` and you can assign a policy to an endpoint, and only the users that meet the policy requirements can access that resource. In this case we are creating a policy that requires users to have the role `admin` and be authenticated.

### Dependency Injection

Dependency Injection is a topic many people might not be too familiar with, but in ASP.NET Core it is fundamental to developing a web application. The concept is simple, you can register services during startup and then call them on any of your controllers, endpoints, even other services. The only requirement for services is that they must all implement an interface.

These services can have different lifecycles, according to our needs. They can be registered as:

- **Singleton**: A singleton will be created only once, the first time it is called. Then, the same instance of the class will be used.
- **Scoped**: Scoped services will be instantiated only once during a request. This means that if more than one service needs the same service
- **Transient**: Transient services are instantiated every time they are requested. Only the light-weightiest services should be implemented in this way.

For more details on dependency injection, check out the [docs](https://docs.microsoft.com/en-us/dotnet/core/extensions/dependency-injection).

## Infrastructure

This Project contains all references to the database and its models. It has 2 folders:

- `Models`: It contains all the database tables defined as classes
- `Migrations`: It contains all the migrations to get the DB up to the current version.

Migrations are essentially classes that will be run to update or downgrade our database to get it to the desired state. These classes will affect the database structure.

### Models

Using Entity Framework, models can be bound to a database table in two ways:

- Using attributes
- Using the fluent API.

For our microservices, we are trying to avoid attributes as much as possible. Let’s say we have the following database model:

```csharp
class Person {
  public int Id { get; set; }
  public string Name { get; set; }
  public string Email { get; set; }
}
```

We can use the `OnModelBinding` method in the class `TEMPLATEDbContext.cs` to define the table properly:

```
protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Person>(
            model =>
            {
                model.HasKey(t => t.Id);
                model.Property(t => t.Id).ValueGeneratedOnAdd();
                model.Property(t => t.Name)
                    .IsRequired()
                    .HasMaxLength(50);
                model.HasIndex(t => t.Email);
                model.Property(t => t.Email).IsRequired();
            }
        );
    }
```

There are a lot of different configurations here, you can check it out in the [docs](https://docs.microsoft.com/en-us/ef/core/modeling/).

### Migrations

Migrations are both restoration points for rollbacks as well as new points in the model configuration. They describe how your database scheme is represented by the data models you defined in the classes. 

If everything is set up properly, migrations should run automatically when the application starts up, but for this we need to create these so called migrations. The migration tool exists separate from the application, and it can be installed with:

```bash
$ dotnet tool install --global dotnet-ef
```

If you are using a Mac with the M1 processor, be sure to add `-a arm64`.

To create a new migration, we can run:

```bash
$ dotnet ef migrations add <MigrationName> 
```

We can also delete migrations. It is important to consider that if a migration has been applied, then it shouldn’t be removed. To remove a migration:

```bash
$ dotnet ef migrations remove
```

To list all migrations:

```bash
$ dotnet ef migrations list
```

And, if we want to delete all migrations, we can just delete the migrations folder.

To apply migrations we have two options. We can either apply all the migrations:

```bash
$ dotnet ef database update
```

Or we can simply apply the migrations up to a certain migration. In this case we can either rollback a migration or move to a newer one:

```bash
$ dotnet ef database update <MigrationName>
```

## Http Client

As part of the development of our Web APIs, we will write our own Http Client for said API. Given the nature of microservices, our clients should support retry strategies and timeouts, to short-circuit requests that take too long to be resolved. This problem is solved using a resilience library called [Polly](http://www.thepollyproject.org/).

### How to Develop

Let’s say we want to write a method to call one of our endpoints. 

The first step is to add the method definition on the `ITEMPLATEClient` interface. Because this project depends on the Web API project, we can (and should) use the endpoint’s Request and Response models on this method definition.

```csharp
public interface ITEMPLATEClient
{
    Task<Response> FooAsync(Request data, CancellationToken token = default);
}
```

Then we have to work on the implementation. We can create a new file with the name `TEMPLATEClient.<METHOD><ACTION>.cs` and implement the method in the partial class `TEMPLATEClient`. Inside this method, we can do whatever logic we want, although it must be limited to our api call.

```csharp
public partial class TEMPLATEClient
{
    public Task<Response> FooAsync(Request data, CancellationToken token = default)
    {
      	... // Do something.
        return CreateRequest<Response>()
            .ExecuteAsync(
                (client, ct) => client.Request("foo")
          														... // Load data into the request
                                      .GetJsonAsync<Response>(ct),
                token
            );
    }
}
```

To execute the API call we must call the function `CreateRequest` and then call the function `ExecuteAsync` with our request built as a lambda parameter. The parameter `client` is an instance of `IFlurlClient` and it comes preloaded with the authorization header and the host. For more information about the Flurl library, check out the [docs](https://flurl.dev/).

### How to Use

Let’s say we want to use any client library in one of our microservices. The first thing about it would be to add the proper configuration in the `appsettings.json` or environment variables. The HttpClient class expects the following configuration variables:

```json
{
  ...
  "TEMPLATEHttpClient": {
    "BaseUrl": "...",
    "Retries": 3,
    "Timeout": 5,
  }
  ...
}
```

Where `BaseUrl` is the base url of the TEMPLATE microservice, `Retries` means the amount of times any HTTP request will be retried if there are any transient errors and `Timeout` means the time it will wait until short-circuiting the call. Both the `Retries` and `Timeout` parameters are optional, and they have 3 and 5 as default values.

The next step is to register the client service using dependency injection. For this we have to go to the file `StartUp/ServiceRegistrator.cs` and add the following:

```c#
internal static class ServiceRegistrator
{
    public static void Register(WebApplicationBuilder builder)
    {
        ...
        builder.Services.AddTEMPLATEHttpClient()
        ...
    }
}
```

Then on our endpoint we can simply use dependency injection to access this service:

```csharp
internal class Endpoint: Endpoint<Request,Response>
{
  public ITEMPLATEHttpClient TEMPLATE { get; set; }
  ...
}
```
























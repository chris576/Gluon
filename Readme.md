# Gluon - The Ktor CQRS Open-Source Plugin

This project brings to life the concept of an open-source CQRS plugin tailored for Ktor. It's not designed to compete
with the AxonIQ + Spring ecosystem or to power large-scale backend monoliths—those are domains where AxonIQ + Spring
excel. Instead, this project fills a gap for smaller-scale applications where lightweight frameworks and resource
efficiency are key considerations.

If you're looking for a simple, flexible CQRS solution that fits snugly into the Ktor ecosystem, this might be exactly
what you need. Check it out on GitHub and join me in refining this approach for developers who value performance and
simplicity.

## Simplifying Controllers and Command Handling

Consider the typical approach in Spring Boot with AxonIQ:

```kotlin
@RestController
class ExampleController(val commandGateway: CommandGateway) {
    @PostMapping("/entity")
    fun doSave(entity: Entity): ResponseEntity<Any> {
        this.commandGateway.sendAndWait<Unit>(entity)
        return ResponseEntity.created(URI("/route")).body("")
    }
}
```

This setup makes sense in large-scale applications, where decoupling HTTP handling from the command bus is essential for
scalability and flexibility. However, in small-scale applications, this level of abstraction can be overkill.

In most scenarios, the HTTP request itself serves as the primary trigger for commands. If 99% of your use cases follow
this pattern, why not simplify? By directly mapping HTTP endpoints to command execution, we reduce boilerplate, improve
readability, and streamline development for lightweight applications.

This project explores how we can refine this process for the Ktor ecosystem, leveraging CQRS principles without
unnecessary complexity.

The typical approach to handling HTTP requests in Ktor is straightforward and functional:

```kotlin
fun Application.configureRouting() {
    routing {
        get {
            val tasks = TaskRepository.allTasks()
            call.respondText(
                contentType = ContentType.parse("text/html"),
                text = tasks.tasksAsTable()
            )
        }
    }
}
```

While this is clean and efficient, it doesn’t inherently provide a seamless way to tie HTTP routes directly to events in
a CQRS architecture. For small-scale applications, where lightweight and expressive solutions are essential, introducing
a domain-specific language (DSL) can bridge this gap.

Here’s an example of how such a DSL could look in practice:

```kotlin
fun CommandContext.registerCommand() {
    
    registerCommandTrigger(
      cause = PostRequest(route = "/", dto = MyDto::class),
      commandType = MyCommand::class,
      // Returns error, if validation of the dto failed.
      dtoValidation = { 
          Error.None
      },
      // It is the dto to be translated.
      dtoTranslation = {
          MyCommand(it.value)
      }
    )
  
    registerCommandValidation(
        commandType = MyCommand::class,
        validation = {
            Error.None
        },
        priority = 1
    )
  
  registerCommandHandling(
      commandType = MyCommand::class,
      translation =  {
          MyEvent(it.value)
      }
  )
}
```

This approach has several advantages:

- Expressiveness: The DSL cleanly maps HTTP routes to event triggers, making the intent of the code more explicit.
- Modularity: By grouping routes and their associated handlers, the structure becomes easier to understand and maintain.
- Seamless Event Integration: The event trigger and handler are defined together, streamlining the CQRS workflow.

This project introduces such a DSL to make working with Ktor even more intuitive for CQRS-based applications. By
reducing boilerplate and enhancing clarity, this approach aligns perfectly with the principles of lightweight, scalable
application design.

## Coupling Event Triggers and State Changes

In many CQRS frameworks, such as AxonIQ, a typical setup involves a separate command handler that interacts with the
domain's aggregate to manage state changes:

```kotlin
@Component
class CreateEntityHandler(val repository: Repository<Entity>) {
    @CommandHandler
    fun handle(createEntityCommand: CreateEntity) {
        repository.newInstance {
            Entity.create(UUID.fromString(createEntityCommand.uuid), createEntityCommand.title)
        }
    }
}
```

While this abstraction is valuable in complex, large-scale systems, it can feel excessive for small-scale applications.
In almost all cases, a command handler simply triggers an event that leads to a state change in the aggregate.

Assuming all application state is encapsulated within aggregates (or a composite of the AggregateContext), we can
simplify this flow by removing commands altogether. Instead, we can couple the event trigger directly to the
corresponding state change using a functional approach.

Here’s an example of how this could look using a domain-specific language (DSL):

```kotlin 
fun AggregateContext.registerDefaultAggregate() {
    registerAggregate(
        state = mutableStateOf("Text"),
        id = randomId
    )
    registerEventHandler(
        eventType = MyEvent::class,
        aggregateId = randomId, 
        handler = { event: MyEvent, aggregate: Aggetate ->
            // DO Event Handling
        }
    )
}
```

How It Works

- Event Trigger: Whenever an event of the specified type is registered in the AggregateContext, the rule automatically
  applies.
- State Change: The state of the aggregate is updated to match the state defined in the event, without the need for a
  separate command.

This approach simplifies the workflow by directly associating events with their corresponding state changes. It reduces
boilerplate and focuses on what matters most in small-scale applications: clarity and efficiency.

This concept is still theoretical, but I’m curious—would this approach benefit anyone else? If you’re intrigued or have
feedback, let’s discuss it further.
---
description: Passing an object the things it depends on from outside, instead of having it create or look them up itself.
---

Dependency injection (DI) is passing a component the objects it needs from the outside, instead of having it create or look them up itself.

```typescript
// Hard-wired: always uses the real client
class SignupService {
  private mailer = new SmtpMailer(process.env.SMTP_URL);
}

// Injected: the caller decides which mailer to use
class SignupService {
  constructor(private mailer: Mailer) {}
}
```

The first version shows the usual symptom. To test signup you need a real mail server, or you must patch a module at runtime, and the test sends real email if configuration goes wrong. With injection, a [unit test](./Unit%20test.md) passes a [test double](./Test%20double.md), and production passes the real implementation. The class depends on an interface and on nothing else.

| Form        | How it is supplied         | When to use                                 |
| ----------- | -------------------------- | ------------------------------------------- |
| Constructor | Argument when created      | Default choice, since nothing can be missing |
| Method      | Argument on each call      | The value varies per call                   |
| Property    | Assigned after creation    | Optional dependencies, framework constraints |

Dependency injection does not require a framework. Wiring objects by hand in one startup location, called the composition root, is often enough. Containers such as those in Spring, NestJS and ASP.NET Core automate this wiring, which helps in large applications but can make it harder to trace where an object came from. Errors may also appear only at startup or at first use.

Containers introduce lifetime bugs. Objects can be created once per application or once per request. If a singleton captures a request-scoped object, it keeps the first request's data, and one user's context can appear in another user's response. Check that long-lived objects do not hold short-lived ones.

A constructor with ten parameters usually indicates a class doing too much, and DI has made that visible. DI is the mechanism that connects adapters to ports in [hexagonal architecture](./Hexagonal%20architecture.md). Simple pure functions and value objects do not need it.

_Usage:_

"I can't test this service without it sending a real email."

"It builds its own mail client. Use dependency injection so the test can pass in a fake."

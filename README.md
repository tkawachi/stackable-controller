## Play2 Stackable Action Composition

This module offers Action Composition Utilities to Play2.1, 2.2 applications


## Target


This module targets the Scala version of Play2.2.x

This module has been tested on Play2.2.1


## Motivation

[Action Composition](http://www.playframework.com/documentation/2.1.0/ScalaActionsComposition) is somewhat limited in terms of composability.

For example, imagine that we want automatic DB transaction management, auth, and pjax functionality.


```scala
def TxAction(f: DBSession => Request[AnyContent] => Future[SimpleResult]): Action[AnyContent] = {
  Action { request =>
    DB localTx { session =>
      f(session)(request)
    }
  }
}

def AuthAction(authority: Authority)(f: User => DBSession => Request[AnyContent] => Future[SimpleResult]): Action[AnyContent] = {
  TxAction { session => request =>
    val user: Either[Result, User] = authorized(authority)(request)
    user.right.map(u => f(u)(session)(request)).merge
  }
}

type Template = Html => Html
def PjaxAction(authority: Authority)(f: Template => User => DBSession => Request[AnyContent] => Future[SimpleResult]): Action[AnyContent] = {
  AuthAction(authority) { user => session => request =>
    val template = if (req.headers.keys("X-Pjax")) views.html.pjaxTemplate.apply else views.html.fullTemplate.apply
    f(template)(user)(session)(request)
  }
}
```

```scala
def index = PjaxAction(NormalUser) { template => user => session => request => 
  val messages = Message.findAll(session)
  Ok(views.hrml.index(messages)(template))
}
```

So far so good, but what if we need a new action that does both DB transaction management and pjax?

We have to create another PjaxAction.

```scala
def PjaxAction(f: Template => DBSession => Request[AnyContent] => Future[SimpleResult]): Action[AnyContent] = {
  TxAction { session => request =>
    val template = if (req.headers.keys("X-Pjax")) html.pjaxTemplate.apply else views.html.fullTemplate.apply
    f(template)(session)(request)
  }
}
```

What a mess!


As an alternative, this module offers Composable Action composition using the power of traits.

## Example

1. First step, Create a sub trait of `StackableController` for every function.

    ```scala
    trait AuthElement extends StackableController with AuthConfigImpl {
        self: Controller with Auth =>

      case object AuthKey extends RequestAttributeKey[User]
      case object AuthorityKey extends RequestAttributeKey[Authority]

      override def proceed[A](req: RequestWithAttributes[A])(f: RequestWithAttributes[A] => Future[SimpleResult]): Future[SimpleResult] = {
        (for {
          authority <- req.get(AuthorityKey).toRight(authorizationFailed(req)).right
          user      <- authorized(authority)(req).right
        } yield super.proceed(req.set(AuthKey, user))(f)).merge
      }

      implicit def loggedIn(implicit req: RequestWithAttributes[_]): User = req.get(AuthKey).get

    }
    ```

    ```scala
    trait DBSessionElement extends StackableController {
        self: Controller =>

      case object DBSessionKey extends RequestAttributeKey[DB]

      abstract override def proceed[A](req: RequestWithAtrributes[A])(f: RequestWithAttributes[A] => Future[SimpleResult]): Future[SimpleResult] = {
        val db = DB.connect()
        db.begin()
        super.proceed(req.set(DBSessionKey, db))(f)
      }

      override def cleanupOnSucceeded[A](req: RequestWithAttributes[A]): Unit = {
        try {
          req.getAs[DB](DBSessionKey).map { db =>
            try {
              db.commit()
            } finally {
              db.close()
            }
          }
        } finally {
          super.cleanupOnSucceeded(req)
        }
      }

      override def cleanupOnFailed[A](req: RequestWithAttributes[A], e: Throwable): Unit = {
        try {
          req.getAs[DB](DBSessionKey).map { db =>
            db.rollbackIfActive()
            db.close()
          }
        } finally {
          super.cleanupOnFailed(req, e)
        }
      }

      implicit def dbSession(implicit req: RequestWithAttributes[_]): DBSession = req.get(DBSessionKey).get.withinTxSession() // throw

    }
    ```

    ```scala
    trait PjaxElement extends StackableController with AuthConfigImpl {
        self: Controller with Auth =>

      type Template = Html => Html

      case object TemplateKey extends RequestAttributeKey[Template]

      override def proceed[A](req: RequestWithAttributes[A])(f: RequestWithAttributes[A] => Future[SimpleResult]): Future[SimpleResult] = {
        val template = if (req.headers.keys("X-Pjax")) views.html.pjaxTemplate else views.html.fullTemplate
        super.proceed(req.set(TemplateKey, template))(f)
      }

      implicit def template(implicit req: RequestWithAttributes[_]): Template = req.get(TemplateKey).get

    }
    ```

2. mix your traits into your Controller

    ```scala
    object Application extends Controller with PjaxElement with AuthElement with DBSessionElement with Auth with AuthConfigImpl {

      def messages = StackAction(AuthorityKey -> NormalUser) { implicit req =>
        val messages = Message.findAll
        Ok(html.messages(messages)(loggedIn)(template))
      }

      def editMessage(id: MessageId) = StackAction(AuthorityKey -> Administrator) { implicit req =>
        val messages = Message.findAll
        Ok(html.messages(messages)(loggedIn)(template))
      }

    }
    ```

3. Mixin different combinations of traits, depending on the functionality that you need.

    ```scala
    object NoAuthController extends Controller with PjaxElement with DBSessionElement {
      
      def messages = StackAction { implicit req =>
        val messages = Message.findAll
        Ok(html.messages(messages)(GuestUser)(template))
      }

      def editMessage(id: MessageId) = StackAction { implicit req =>
        val messages = Message.findAll
        Ok(html.messages(messages)(GuestUser)(template))
      }

    }
    ```

## How to use

Add a dependency declaration into your Build.scala or build.sbt file:

```scala
libraryDependencies += "jp.t2v" %% "stackable-controller" % "0.3.0"
```

## ExecutionContext

When you want to use `scala.concurrent.ExecutionContext`, you can use `StackActionExecutionContext` method.

`StackActionExecutionContext` returns an `ExecutionContext` that is given when `StackAction` method calling.
So, users of your StackElement can customize `ExecutionContext`.

```scala
    trait AsyncElement extends StackableController with AuthConfigImpl {
        self: Controller with Auth =>

      private case object FooKey extends RequestAttributeKey[Foo]

      override def proceed[A](req: RequestWithAttributes[A])(f: RequestWithAttributes[A] => Result): Result = {
        val ctx: ExecutionContext = StackActionExecutionContext(req)
        val future: Future[Foo] = getFooAsynchronously(ctx)
        Async {
          future map { 
            foo => super.proceed(req.set(FooKey, foo))(f)
          } recover {
            _ => super.proceed(req)(f)
          }
        }
      }

      implicit def foo(implicit req: RequestWithAttributes[_]): Foo = req.get(FooKey).get

    }
```


## License

This library is released under the Apache Software License, version 2, which should be included with the source in a file named `LICENSE`.

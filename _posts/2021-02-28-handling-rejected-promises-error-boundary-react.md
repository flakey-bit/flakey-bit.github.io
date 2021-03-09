---
title: 'Handling rejected promises in React with an error boundary'
date: 2021-02-28T07:15:17+13:00
author: eddiewould
layout: post
permalink: /2021/28/28/handling-rejected-promises-error-boundary-react/
spay_email:
  - ""
categories:
  - Uncategorized
---

Treat unhandled promise rejections (exceptions in `async` code) consistently with regular synchronous exceptions in React.

### Error Boundaries
In React, an error boundary is a component responsible for trapping unhandled exceptions & responding appropriately (e.g. logging to an API and/or rendering a friendly error page for the user).

Error Boundaries are typically near the top of the component tree (in the same way that a console app might catch unhandled exceptions in its `main()` method).

Because (at time of writing!) there is no equivalent hook for React's `getDerivedStateFromError`, Error Boundaries must be implemented as class-based components (rather than function-based).

From the [React docs](https://reactjs.org/docs/error-boundaries.html), their structure is approximately: 

```react
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    // Update state so the next render will show the fallback UI.
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    // You can also log the error to an error reporting service
    logErrorToMyService(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      // You can render any custom fallback UI
      return <h1>Something went wrong.</h1>;
    }

    return this.props.children; 
  }
}
```

### Background on Promises
If you know all about Promises already, feel free to skip this section (& the following section on `async` / `await`).

> A Promise represents the eventual result of an asynchronous operation.

An asynchronous operation is a task you start without "standing around" waiting for it to complete. In other words, after starting the task code execution resumes _immediately_ - regardless of how long the task _actually_ takes to complete. 

There are of course cases where the asynchronous operation is "fire and forget", but usually the application will perform further processing when the task eventually completes (successfully or otherwise).

An example of an asynchronous operation that most web developers are familiar with is a "xhr" (aka "AJAX") request.

A Promise is an abstraction that makes dealing with asynchronous operations much simpler (and more consistent). Before Promises were introduced around 2014, asynchronous code was typically handled in JavaScript by continuation-passing.

Consider an asynchronous operation that produces data of type `T` (e.g. a `GET` request to a server that returns a `UserProfile` DTO). With a promise-based API, calling code will invoke the asynchronous operation by calling a function - rather than returning the result `T` directly, the function returns an object (a Promise). 

The Promise object has method named `then()` which takes a couple of arguments, both callback functions:
* `onSuccess`[^1]: Function to be called (with the data of type `T`) when the asynchronous operation completes successfully. The function may return either
  * `null` or `undefined`
  * Data of some `T1` (i.e. by transforming the `T` it receives into a `T1`)
  * A _Promise_ for for data of type `T1` (e.g. by calling another function that returns a Promise)
* `onRejected`: Function to be called (with a *rejection reason* i.e. error) when the asynchronous operation fails. Similarly to `onSuccess`, the function may return either
  * `null` or `undefined`
  * Data of type `T2`. This might be derived from the rejection reason somehow, or it could be a default value
  * A _Promise_ for for data of type `T2` (e.g. by calling another function that returns a Promise).
Note that `T1` & `T2` could potentially be the same type.

[^1]: `onSuccess` is sometimes referred to as `onfulfilled`.

Some important things to know about Promises:
* Promises may resolve synchronously (immediately). This doesn't need to concern the calling code (which is great!)
* Promises are intended to be chained. The return value from calling `then` is always another (*new*) Promise.
* If a Promise is rejected then downstream promises in the chain will be rejected too.
* If the reference to a Promise is saved, it is possible to create multiple independent chains from the resolution.
  ```javascript
      const somePromise = doSomethingThatReturnsAPromise();
      
      somePromise
        .then(computeAverage)    // new promise
        .then(logAverage);       // new promise

      somePromise
        .then(computeMaximum)    // new promise
        .then(renderMaxValue);   // new promise
  ```

For completeness, here is the TypeScript definition for `then`:
```typescript
interface PromiseLike<T> {
    /**
     * Attaches callbacks for the resolution and/or rejection of the Promise.
     * @param onfulfilled The callback to execute when the Promise is resolved.
     * @param onrejected The callback to execute when the Promise is rejected.
     * @returns A Promise for the completion of which ever callback is executed.
     */
    then<TResult1 = T, TResult2 = never>(onfulfilled?: ((value: T) => TResult1 | PromiseLike<TResult1>) | undefined | null, onrejected?: ((reason: any) => TResult2 | PromiseLike<TResult2>) | undefined | null): PromiseLike<TResult1 | TResult2>;
}
```

Shrewd readers will notice the glaring omission of `catch`. In reality, `catch` is really just a convenience method/ thin-wrapper for `then`:
* It registers a `onRejected` callback for rejection of the Promise
* For `onSuccess`, the "identity" function is used i.e. given data `T` return that same data.

A common usage of `catch` is to handle rejection right at the end of the promise-chain (regardless of which step the problem occurred) - used in this way it better conveys the intent. This [post](https://stackoverflow.com/a/24663315/855208) describing the differences between `.catch` and supplying a `onRejected` callback.

More resources on JavaScript Promises:
* [promisesaplus.com](https://promisesaplus.com/)
* [developer.mozilla.org](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises)

#### await/async
I almost didn't bother mentioning these (again, better resources elsewhere). 

*TL;DR*: (in JavaScript) `async` / `await` are essentially syntactic-sugar that allow you to write asynchronous code that _looks_ (more) like regular synchronous JavaScript.

For example, this function written using Promises

```typescript
const fetchById = (entityId: EntityId) : Promise<EntityDto | null> => {
  return apiClient
      .get<EntityDto>(`v1/entity/${entityId}`)
      .then((response) => response.data)
      .catch(error => null); // 'error' parameter is rejection-reason for Promise
}
```

could be rewritten using `async` + `await` as:

```typescript
const fetchById = async (entityId: EntityId) : Promise<EntityDto | null> => {
  try {
    const response = await apiClient.get<EntityDto>(`v1/entity/${entityId}`);
    return response.data;
  } catch (error) {
    return null;
  }
}
```

#### Error / rejection typing

It's worth remembering that exception-handling remains a mess in JavaScript, regardless of which approach (`async` / `Promise`) you take:
- The rejection reason (passed to the `onRejected` callback) is of type `any`
- The 'exception' in a `catch` block is of type `any`

Basically, you can throw pretty much anything as an exception, so you need to be prepared to catch pretty much anything (strings included). That means `instanceof` checks are needed (yes, in 2021) 🤮. 

### The problem
For exceptions in regular _synchronous_ code, the ErrorBoundary (beginning of this post) works really well. 

For example, during rendering your code might try to call a method on `null` object - the JavaScript runtime will throw a `TypeError` and that will be caught by the React ErrorBoundary (which would show a friendly error page to the user).

However for _asynchronous_ code (involving Promises) if the promise-chain doesn't handle rejection then *the user may have no idea something went wrong*. This is particularly true with fire-and-forget asynchronous operations.

In other words, if your application 
* Invokes a method returning a Promise but doesn't call `then` supplying a `onRejected` callback (or use `catch`)
* Calls an `async` method without using `await`
* Calls an `async` method, uses `await` but doesn't catch exceptions

it is open to unhandled promise rejections. 

The problem with unhandled promise rejections is that the user may not *know* that something went wrong. Depending on what went wrong and your application semantics, that could be significant. 

I'd argue it's best practice to ensure all Promise rejections are handled and *to treat unhandled promise rejections as a coding error*.

### Solution
With some modifications, we can handle (otherwise unhandled) promise rejections (globally) and plumb them through to our `ErrorBoundary`.

When our `ErrorBoundary` mounts/unmounts, it registers/unregisters (respectively) an event handler for promise rejection. 

The event handler takes the rejection reason and uses it to update the state for the `ErrorBoundary`.

With this approach, the `ErrorBoundary` works consistently regardless of whether the exception occurred in synchronous code or in asynchronous code - we can show the same friendly error page to the user.

```react
interface State {
    error: any; // Could be an exception thrown in synchronous code or could be a rejection reason from a Promise, we don't care
}

class ErrorBoundary extends Component<State> {
    private promiseRejectionHandler = (event: PromiseRejectionEvent) => {
        this.setState({
            error: event.reason
        });
    }

    public state: State = {
        error: null
    };

    public static getDerivedStateFromError(error: Error): State {
        // Update state so the next render will show the fallback UI.
        return { error: error };
    }

    public componentDidCatch(error: Error, errorInfo: ErrorInfo) {
        console.error("Uncaught error:", error, errorInfo);
    }

    componentDidMount() {
        // Add an event listener to the window to catch unhandled promise rejections & stash the error in the state
        window.addEventListener('unhandledrejection', this.promiseRejectionHandler)
    }

    componentWillUnmount() {
        window.removeEventListener('unhandledrejection', this.promiseRejectionHandler);
    }

    public render() {
        if (this.state.error) {
            const error = this.state.error;

            let errorName;
            let errorMessage;

            if (error instanceof PageNotFoundError) {
                errorName = "...";
                errorMessage = "..."
            } else if (error instanceof NoRolesAssignedError) {
                errorName = "...";
                errorMessage = "..."
            } else {
                errorName = "Unexpected Application Error";
            }

            return <FriendlyError errorName={errorName} errorMessage={errorMessage} />
        } else {
            return this.props.children;
        }
    }
}
```

Hopefully you find this technique useful. Would be interested to know your thoughts / how you've tackled this problem in your applications.
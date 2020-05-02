# Design Pattern #3: Extracting API Response Handlers Out of Store Actions

## Introduction

In idiomatic Vue, all API/backend interactions are typically relegated to store actions. This is a very good way to keep the point where we interact with the backend data in one comfortable place from where we can make modifications as and when things change in the backend responses.

As part of this, developers write a lot of `fetch`-like calls in applications that use a RESTful API. And as you can see where we're going with this, that's a lot of repetition.

Think about the things a `fetch` (or a `fetch`-like call) has to do/handle:

- it has to call an endpoint/URL with specific parameters for the HTTP method, headers, request data, query params etc.
- once the backend responds, it has to handle the response. If successful response, it has to process the data and hand it over to the function or expression that called it. If error, it has to process the error and perhaps add some more information (like status) etc.

All of these things can be generalized for most API calls. And let's look at how we can make that possible.

Note: these things, in a strict sense, fall outside Vue's ecosystem of design patterns. However, they fit neatly into the idea of using store actions to run API calls.

## Extracting Fetch

Here's an example of a store actions object that calls two different endpoints:

```js
actions: {
    async getUserInfo(ctx, payload) {
      ctx.commit("fetchUserInfoStart")
      try {
        const a = await fetch(
          `https://jsonplaceholder.typicode.com/users/${payload.id}/`
        ).then((r) => {
          if (r.ok) {
            return r.json();
          } else {
            throw new Error(r.status)
          }
        });
        ctx.commit("fetchUserInfoSuccess", a);
      } catch (e) {
        ctx.commit("fetchUserInfoFailed", e)
      }
      ctx.commit("fetcUserInfoEnd")
    },
    async getAlbums(ctx, payload) {
      ctx.commit("fetchAlbumsStart")
      try {
        const a = await fetch(
          `https://jsonplaceholder.typicode.com/users/${payload.id}/albums`
        ).then((r) => {
          if (r.ok) {
            return r.json();
          } else {
            throw new Error(r.status)
          }
        });
        const formatted = a.map(d => ({ id: d.id, title: d.title }))
        ctx.commit("fetchAlbumsSuccess", formatted);
      } catch (e) {
        ctx.commit("fetchAlbumsFailed", e)
      }
      ctx.commit("fetchAlbumsEnd")
    }
}
```

In both the actions, we are calling the API endpoint (via `fetch`) and then processing the response. If everything went ok (`r.ok` refers to any response with status code in the 2xx range), we process the data and send it to a mutation in the store. Otherwise, we throw an error which sends the error status to another set of mutations.

We are definitely repeating something there: the actual `fetch` calls. If you use any other library (like `axios` for example), you'd be doing the same.

If you notice, each action appears to be running some sort of a request to an API. This could be of any HTTP method, and can contain any set of request parameters or data (for eg in the case of a POST request). 

Instead of writing a lengthy `fetch` (or `axios`) method in every action, we can reduce the complexity to something like this:

```js
actions: {
    async getUserInfo(ctx, payload) {
      ctx.commit("fetchUserInfoStart")
      const response = SomeHelperFunction({ url: `https://jsonplaceholder.typicode.com/users/${payload.id}/`})
      if (response.error) {
        ctx.commit("fetchUserInfoFailed", response.error)
      }
      if (response.data) {
        ctx.commit("fetchUserInfoSuccess", response.data)
      }
      ctx.commit("fetcUserInfoEnd")
    },
    async getAlbums(ctx) {
      ctx.commit("fetchAlbumsStart")
      const response = SomeHelperFunction({ url: `https://jsonplaceholder.typicode.com/users/${payload.id}/albums`})
      if (response.error) {
        ctx.commit("fetchAlbumsFailed", response.error)
      }
      if (response.data) {
        ctx.commit("fetchAlbumsSuccess", response.data)
      }
      ctx.commit("fetchAlbumsEnd")
    }
}
```

Here, `SomeHelperFunction` is the abstract we are looking for. 

You will notice that the output from that function appears to have two keys that we're interested in: `data` and `error`. After all, an API call is going to either succeed or throw an error.

Let's write the `SomeHelperFunction` function!

The objective of this function is to do a few things:
- take the URL to call
- know which method to call (GET/PUT/POST etc)
- accept query parameters optionally
- accept request data optionally
- run the request
- return the response as an object of the shape `{ data: any | null, error: any | null }`

If we are able to do this, our store actions need to simply call this function and then wait for a response. In the response, all we need to do is look for `data` or `error`. If `error` is not null, we know the request errored out for some reason (which is also contained in the `error` as an object).

```js
// SomeHelperFunction a.k.a CustomFetch
export default async ({
    url,
    method="GET",
    params=null,
    data=null
} = {}) => {
    if (!url) return { data: null, error: { message: "No url given to run the request" }}
    const URL = params ? addParamsToURL(url) : url
    // let us assume "addParamsToURL" is a function we have already that converts params object into key=value and attaches to the URL.

    let response = { data: null, error: null }
    try {
        const res = await fetch(URL, {
            method,
            ...(data && { body: JSON.stringify(data) })
        }).then(r => {
            return r.ok ? { data: r.json(), error: null } : { data: null, error: { status: r.status, message: "Request failed with non-2xx status", data: r.json() } }
        })
        response = { ...res }
    } catch (e) {
        response = { data: null, error: { data: e, message: "Something went wrong" }}
    }
    return response
}
```

The `CustomFetch` (a.k.a `SomeHelperFunction`) calls the fetch for the URL based on the options given and then returns a guaranteed structure of an object as response. This structure has `data` and `error` each of which may be either null or something (like an object or array). The `error`, furthermore, is guaranteed to have a `error.message` (which is a string) and `error.data` (which could be anything, generally contains the object that has more information about the error).

There are three major benefits to this approach:

1. Your store actions become concise
2. Handling errors coming out of the actual `fetch` (or due to any other reason) is no longer an issue
3. Most importantly, you have a standard interface of response coming out of your CustomFetch helper so you can always rely on checking for `response.data` or `response.error` to know the status of your API call.

The same thing can be replicated for any promise-based library used for calling the API like `axios`.

## Extending The Library With Definitions

Now that we've extracted the main function that calls the API and gets us data (or error), we can do a few extra things with it.

Most developers try to put all their API endpoints (and related assets) in a single module/file so that it is easier to handle API-related things.

For instance, you might put all your endpoints in a single constant (like an enum) so it becomes easy to use and even construct new endpoints from old ones (composition).

In our previous example, we pass the endpoint / API URL as a string. This, however, does not reflect real world where these endpoints are typically stored as constants (or functions).

As an example, let's take the case of a small content application with users and their todos.

In this example, we have these endpoints and methods:

```
Base URL = https://jsonplaceholder.typicode.com

1. Get all users
GET /users

2. Get all todos of a specific user
GET /todos?userId={userId}

3. Create a new todo for a user
POST /todos
{ userId: int, title: string, completed: boolean }

4. Mark a todo as complete
PATCH /todos/:id
{ title: string, completed: boolean }

5. Delete a todo
DELETE /todos/:id
``` 

Remember our CustomFetch helper function? In the argument it takes, we can send it a few things including the URL, the method type and request body/data.

Suppose we want to get all the todo of a specific user (userId = 1), we would write an action like this:

```js
async getTodosForUser(ctx, payload) {
    ctx.commit("fetchTodosStart")
    const response = await CustomFetch({
        url: "https://jsonplaceholder.typicode.com" + "/todos/" + payload.userId
    })
    if (response.error) {
        ctx.commit("fetchTodosFailed", response.error)
    }
    if (response.data) {
        ctx.commit("fetchTodosSuccess", response.data)
    }
    ctx.commit("fetchTodosEnd")
}
```

And to update a todo (marking it as complete), we'd have an action that looks like this:

```js
async updateTodo(ctx, payload) {
    ctx.commit("updateTodoStart")
    const response = await CustomFetch({
        url: "https://jsonplaceholder.typicode.com" + "/todos/" + payload.todoId,
        method: 'PATCH',
        data: {
            completed: true
        }
    })
    if (response.error) { return ctx.commit("updateTodoFailed", response.error)}
    if (response.data) { return ctx.commit("updateTodoSuccess", response.data)}
    ctx.commit("updateTodoEnd")
}
```

We can simplfy these things a little further.

For instance:

- some endpoints have multiple methods but the only difference is which action calls it.
- it shouldn't be necessary to build the whole URL every time. URLs should be composable (or generated) automatically

We can use a simple dictionary to generate URL and Method pairs for us which can then be used in CustomFetch easily. It may sound trivial to do this but it becomes far easier to make store actions read like a story of what's happening once we have this in place.

Let's take the case of our todo example.

First, we're going to describe a dictionary that can generate URL and method pairs whenever we call it.

Remember that in order to describe this dictionary, we're making use of this:

```
Base URL = https://jsonplaceholder.typicode.com

1. Get all users
GET /users

2. Get all todos of a specific user
GET /todos?userId={userId}

3. Create a new todo for a user
POST /todos
{ userId: int, title: string, completed: boolean }

4. Mark a todo as complete
PATCH /todos/:id
{ title: string, completed: boolean }

5. Delete a todo
DELETE /todos/:id
```

```js
// FetchDictionary.js
const BASE_URL = "https://jsonplaceholder.typicode.com"
export default {
    getUsers: () => {
        return {
            url: BASE_URL + "/users",
            method: "GET"
        }
    },
    getUserTodos: (userId) => {
        return {
            url: BASE_URL + "/todos/?userId=" + userId,
            method: "GET"
        }
    },
    createNewTodo: () => {
        return {
            url: BASE_URL + "/todos",
            method: "POST"
        }
    },
    updateTodo: (todoId) => {
        return {
            url: BASE_URL + "/todos/" + todoId,
            method: "PATCH"
        }
    },
    deleteTodo: (todoId) => {
        return {
            url: BASE_URL + "/todos/" + todoId,
            method: "DELETE"
        }
    }
}
```

Now, in our store actions, we don't have to explicitly pass on the URL and method in the CustomFetch helper. Instead, we just use the keys from this library and that helps us generate the URL and Method pairs to be used in CustomFetch.

And when you look at the store actions now, you'll notice that by hiding away the internal details of the URL and other options, it reads easier.

```js
import FetchDictionary from './FetchDictionary.js'
actions: {
    // get users
    async getUsers(ctx, payload) {
        ctx.commit("fetchUsersStart")
        const response = await CustomFetch({
            ...FetchDictionary.getUsers(payload.userId)
        })
        // ... rest of the function
    },
    async getTodosForUser(ctx, payload) {
        ctx.commit("fetchTodosStart")
        const response = await CustomFetch({
            ...FetchDictionary.getUserTodos(payload.userId)
        })
        // ... rest of the function
    },
    async createNewTodo(ctx, payload) {
        ctx.commit("createTodoStart")
        const response = await CustomFetch({
            ...FetchDictionary.createNewTodo(),
            data: {
                userId: payload.userId,
                title: payload.todoTitle,
                completed: payload.status || false
            }
        })
        // ... rest of the function
    },
    async updateTodo(ctx, payload) {
        ctx.commit("updateTodoStart")
        const response = await CustomFetch({
            ...FetchDictionary.updateTodo(payload.todoId),
            data: {
                title: payload.todoTitle,
                completed: payload.status
            }
        })
        // ... rest of the function
    }
}
```

By abstracting away the URLs and the methods, we get a cleaner store actions suite that can show us what data is actually being passed around.

However, there is still one more thing that we can achieve here. Let's look at that in the next section.

## Using Transformations In Definitions

In our last example, notice these two actions:

```js
// ...
async createNewTodo(ctx, payload) {
    ctx.commit("createTodoStart")
    const response = await CustomFetch({
        ...FetchDictionary.createNewTodo(),
        data: {
            userId: payload.userId,
            title: payload.todoTitle,
            completed: payload.status == 'done' ? true : false
        }
    })
    // ... rest of the function
},
async updateTodo(ctx, payload) {
    ctx.commit("updateTodoStart")
    const response = await CustomFetch({
        ...FetchDictionary.updateTodo(payload.todoId),
        data: {
            title: payload.todoTitle,
            completed: payload.status === 'done' ? true : false
        }
    })
    // ... rest of the function
}
```

See how we're not directly passing on `payload` into `data` of `CustomFetch` but instead, we're transforming the keys? This is because the API wants the data in a specific shape but the component/page that is calling the store action is actually generating the same data in a different shape.

This is not uncommon when teams - frontend and backend - develop components and API spec independently and the frontend team does not stick to the same keys as the API spec when designing pages or components. In fact, it is recommended that the frontend team not depend or use the same properties consumed/produced by the API spec.

There are a lot of scenarios when it is good for the frontend team to stick to its own set of (or shape of) stateful data when designing components or pages. When the API is in flux, or when there are big changes in the API spec, you shouldn't have to hunt down every component making use of the data and change the keys accordingly. Instead, one single place of transformation should take care of that.

Since we already have something like a dictionary that generates URL and methods, we can also attach transformation functions to this. Our store actions will then only be doing two things when calling `CustomFetch`:

1. call the right function from `FetchDictionary`
2. send the payload data as it is

To help you visualize how the store actions might look, here they are:

```js
// ...
async createNewTodo(ctx, payload) {
    ctx.commit("createTodoStart")
    const response = await CustomFetch({
        ...FetchDictionary.createNewTodo(),
        data: payload.data
    })
    // ... rest of the function
},
async updateTodo(ctx, payload) {
    ctx.commit("updateTodoStart")
    const response = await CustomFetch({
        ...FetchDictionary.updateTodo(payload.todoId),
        data: payload.data
    })
    // ... rest of the function
}
```

Let's take the case of creating a new todo.

In our `FetchDictionary`, we'll define a function that will describe how `payload.data` should be transformed into something that the API can accept.

```js
// FetchDictionary.js
export default {
    // ... other code
    getUserTodos: (userId) => {
        return {
            url: BASE_URL + "/todos/?userId=" + userId,
            method: "GET",
            outputTransform: (res) => {
                if (!res || !res.length) return []
                return res.map(d => ({
                    todoTitle: d.title,
                    userId: d.userId,
                    id: d.id,
                    status: d.completed ? 'done' : 'pending'
                }))
            }
        }
    },
    createNewTodo: () => {
        return {
            url: BASE_URL + "/todos",
            method: "POST",
            inputTransform: (data) => {
                return {
                    title: data.todoTitle,
                    userId: data.userId,
                    completed: d.status === 'done' ? true : false
                }
            }
        }
    },
    updateTodo: (todoId) => {
        return {
            url: BASE_URL + "/todos/" + todoId,
            method: "PATCH",
            inputTransform: (data) => {
                return {
                    title: data.todoTitle,
                    completed: d.status === 'done' ? true : false
                }
            }
        }
    },
    deleteTodo: (todoId) => {
        return {
            url: BASE_URL + "/todos/" + todoId,
            method: "DELETE"
        }
    }
}
```

As you can see, we included an `inputTransform` function. 

In our store actions, since we're simply destructuring the return object `FetchDictionary.method`, our `CustomFetch` will have an additional properties called `inputTransform` and `outputTransform` in the object it receives.

The `CustomFetch` helper will use these transformer functions to re-shape the data going out to the API and data coming in from the API.

Let's tweak `CustomFetch` to use these transform functions.

```js
export default async ({
    url,
    method="GET",
    params=null,
    data=null,
    inputTransform=null,
    outputTransform=null
} = {}) => {
    if (!url) return { data: null, error: { message: "No url given to run the request" }}
    const URL = params ? addParamsToURL(url) : url
    // let us assume "addParamsToURL" is a function we have already that converts params object into key=value and attaches to the URL.

    let response = { data: null, error: null }
    try {
        const res = await fetch(URL, {
            method,
            ...(data && { body: JSON.stringify(
                    inputTransform ? inputTransform(data) : data
                ) })
        }).then(r => {
            return r.ok ? { data: outputTransform ? outputTransform(r.json()) : r.json(), error: null } : { data: null, error: { status: r.status, message: "Request failed with non-2xx status", data: r.json() } }
        })
        response = { ...res }
    } catch (e) {
        response = { data: null, error: { data: e, message: "Something went wrong" }}
    }
    return response
}
```

The changes are subtle.

- First, we add `inputTransform` and `outputTransform` as expected keys in the argument (as mentioned before)
- Next, instead of `JSON.stringify(data)`, we make it `JSON.stringify( inputTransform ? inputTransform(data) : data)`. That is, if there is an `inputTransform` function, we use that to first transform data into a convenient format as specified by the `inputTransform` function and then pass it on. If there is no `inputTransform` function in the argument object, we just skip and pass data directly to the stringify function.
- Similar to the `inputTransform`, we do a check on `outputTransform` before we set the `data` key in the final response. If `outputTransform` exists, we transform the response JSON `r.json()` and pass it to the data. Otherwise, simply pass `r.json()` to data.

And with that, we come to the end of this chapter.

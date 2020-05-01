# Design Pattern #3: Extracting API Response Handlers Out of Store Actions

## Introduction

In idiomatic Vue, all API/backend interactions are typically relegated to store actions. This is a very good way to keep the point where we interact with the backend data in one comfortable place from where we can make modifications as and when things change in the backend responses.

As part of this, developers write a lot of `fetch`-like calls in applications that use a RESTful API. And as you can see where we're going with this, that's a lot of repetition.

Think about the things a `fetch` (or a `fetch`-like call) has to do/handle:

- it has to call an endpoint/URL with specific parameters for the HTTP method, headers, request data, query params etc.
- once the backend responds, it has to handle the response. If successful response, it has to process the data and hand it over to the function or expression that called it. If error, it has to process the error and perhaps add some more information (like status) etc.

All of these things can be generalized for most API calls. And let's look at how we can make that possible.

Note: these things, in a strict sense, fall outside Vue's ecosystem of design patterns. However, they fit neatly into the idea of using store actions to run API calls.

## Extracting Fetch Away to Handle Errors Gracefully

Here's an example of a store actions object that calls two different endpoints:

```js
actions: {
    async getUserInfo(ctx) {
      ctx.commit("fetchUserInfoStart")
      try {
        const a = await fetch(
          "https://jsonplaceholder.typicode.com/users/1/"
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
    async getAlbums(ctx) {
      ctx.commit("fetchAlbumsStart")
      try {
        const a = await fetch(
          "https://jsonplaceholder.typicode.com/users/1/albums"
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
    async getUserInfo(ctx) {
      ctx.commit("fetchUserInfoStart")
      const response = SomeHelperFunction({ url: "https://jsonplaceholder.typicode.com/users/1/"})
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
      const response = SomeHelperFunction({ url: "https://jsonplaceholder.typicode.com/users/1/albums"})
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

## Extending The Library With Definitions
- url and methods appear frequently: can we extract them? 
- replacing that in the library above

## Using Transformations In Definitions
- highlight transformations
- good use cases: 1) API in flux, 2) component state names decoupled from API keys

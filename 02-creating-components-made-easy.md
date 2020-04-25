# Design Pattern #1: Creating Components Made Easy

## Introduction

Most of your time as a Vue.js developer is spent creating (or extending) components. If you work on a project that just got started, you'll most likely be creating new components all day long.

Some components are simple, some are sophisticated beasts. But most components – no matter how simple or sophisticated – have a small part of shared logic: data states and one data-getter function that runs on the `mounted()` or `created()` hooks.

If we take a component that renders a UI based on data, we automatically have three states for this component:

- a loading state to show while the data is fetched
- an error state in case there was some issue with the data fetching
- a data state (which can be split into two: no data and data)

In the wild, the states we talked about above look like this in a component:

```vue
<template>
    <!-- UI code that shows loading spinners, error messages and data. -->
</template>
<script>
export default {
    data() {
        return {
            loading: true,
            data: [],
            error: ''
            ... // other properties
        }
    },
    mounted() {
        // async call to get data and feed it in this.data
        // this async call will update loading and error too
    }
}
<script>
```

Clearly, we're repeating some logic in every new component. We should be able to write lesser code to handle these states every time we create a new component.

Here's how a new component _can_ look:

```vue
<template>
    <DataProvider :run="run">
        <div slot-scope="{ data, loading, error }">
            <template v-if="loading">Loading...</template>
            <template v-if="error">Something went wrong. {{ error }}</template>
            <template v-if="data">
                <!-- show user info here -->
            </template>
        </div>
    </DataProvider>
</template>
<script>
export default {
    methods: {
        async run() {
            // an async function/promise that returns some data/error (like an API call)
        }
    }
}
</script>
```

Or like this:

```vue
<template>
    <DataProvider :run="run">
        <template v-slot:loading>Loading...</template>
        <template v-slot:error="error">Something went wrong. {{ error.message }}</template>
        <template v-slot:data="data">
            <!-- show data here using `data` -->
        </template>
    </DataProvider>
</template>
<script>
export default {
    methods: {
        async run() {
            // an async function/promise that returns some data/error (like an API call)
        }
    }
}
</script>
```

Don't worry if the syntax looks unfamiliar or convoluted. It will take you just a few minutes to understand this.

But do notice that in the `<script>` block, we've almost gotten rid of everything.

## Use Case / Example

Let's take the case of this simple user-info card. It shows an avatar, a name and some other information of a specific user. We get this user by making an API call (in this case, to `https://randomuser.me/api/1.0/?seed=foobaz`). (Just to ensure you understand the properties used in the upcoming code, [look at the response of that call here](https://randomuser.me/api/1.0/?seed=foobar).)

Here's how the card looks:

![User Info Card](https://i.imgur.com/yv2XjJX.png)

Barring styling information, you might end up with a code that looks like this:

```vue
<template>
    <div class="card" v-if="loading">Loading...</div>
    <div class="card" v-else-if="data">
        <div class="avatar">
            <img :src="data.results[0].picture.medium" />
        </div>
        <div class="user-info">
            <div class="name">
                {{ data.results[0].name.first }}
                {{ data.results[0].name.last }}
            </div>
            <div class="email">{{ data.results[0].email }}</div>
            <div class="registered">
                Member since
                {{
                new Intl.DateTimeFormat("en-GB", {
                day: "numeric",
                month: "short",
                year: "numeric",
                }).format(new Date(data.results[0].registered))
                }}
            </div>
        </div>
    </div>
    <div v-else-if="error" class="card">Something went wrong.</div>
</template>
<script>
import axios from "axios";
export default {
    data: () => ({
        data: null,
        loading: true,
        error: null
    }),
    mounted() {
        this.loading = true;
        this.error = null;
        axios
            .get("https://randomuser.me/api/1.0/?seed=foobar")
            .then(r => {
                this.data = r.data;
                this.loading = false;
            })
            .catch(e => {
                this.error = e.response;
                this.loading = false;
            });
    }
};
</script>
```

## Renderless Components and Scoped Slots

Did you notice that there are some things here that could be quite common in most other components you create?

- making an API call to get data
- setting the loading state to true/false depending on the API call resolution
- updating data and error values once the API call resolves

If only there was a way to "outsource" these and get back just a reactive "data", "loading" and "error" information, we can swap them in our code and make it simpler.

Luckily for us, Vue.js has two interesting concepts that can help us here.

- The first is all about slots which you must have either used or heard of.
- The second is what's called "renderless components" which simply work like a component that renders whatever it is given as HTML inside it.

### Slots and Slot Scope

Let's say you have a component, called `DataProvider.vue`, like this:

```vue
<template>
    <div>
        <slot v-bind="user" />
    </div>
</template>
<script>
export default {
    data() {
        return {
            user: { name: 'Becky Smith', email: 'becky.sims@example.com' }
        }
    }
}
</script>
```

We can use `DataProvider` in a new component this way:

```vue
<template>
    <DataProvider>
        <div slot-scope="user">
            Name: {{ user.name }} <br />
            Email: {{ user.email }}
        </div>
    </DataProvider>
</template>
<script>
export default {
    components: {
        DataProvider: () => import('./DataProvider.vue')
    }
}
</script>
```

`DataProvider` provides a "slot" which any component using it can use it like how we use a regular HTML tag. That is, simply put all the content you need to show inside `<DataProvider> ... </DataProvider>`.

Notice that `DataProvider` has a `user` object (a state variable). And not just that, it also exposes that value as-is to whoever us using the `DataProvider` component.

It does this by something called a `slot-scope`.

Two things are happening here:
- Inside `DataProvider` component, we're binding the `user` state object to the default slot (`<slot v-bind="user" />`) - just like we pass props to a component
- Then, in the component using `DataProvider`, we mention a `slot-scope` parameter that picks up whatever data has been exposed by `DataProvider` in the default slot. (In our case, the `user` object. We do this by writing `<div slot-scope="user">`)

There's one hiccup though. 

Notice that the `DataProvider` component adds a `<div>` wrapper to the `<slot>` where our actual component content gets rendered. This means there will be an extra `<div>` wrapper. Not exactly a big issue but let's see if we can get rid of that.

In case you're thinking about it, we can't do this:

```vue
<template>
    <slot v-bind="user" />
</template>
...
```

This is because Vue needs you to define a rendering node (`<template>` and `<slot>` are not part of that definition)

So we look for something else.

### Render Function

In Vue, instead of writing a separate HTML section wrapped in `<template>` you can also provide a `render()` function inside the component options (this is similar to most frontend frameworks like React). This is going to help us avoid the extra `<div>` wrapper and also add some interesting capabilities.

Here's how we can do that for our `DataProvider` component:

```vue
<script>
export default {
    data: () => ({
        user: { name: 'Becky Smith', email: 'becky.sims@example.com' }
    })
    render() {
        return this.$scopedSlots.default({
            user: this.user
        })
    }
}
</script>
```

You will notice a few things here:

- the component now completely gets rid of the `<template>` block
- we have a new `render()` function that `returns` something. According to Vue's rules, the `render()` function should return exactly one VNode (ie we cant return an array)
- we use something called `this.$scopedSlots.default()` to create the VNode and return it from the `render()` function
- and finally, inside our `this.$scopedSlots.default()` function, we pass the data that we need to pass to the slot. In this case, it's the `user` object from the component's data function (that's why we use `this.user`)

<!-- TODO: write an explanation of $slots & $scopedSlots -->

Given this, let us now construct our `DataProvider` component.

These are our basic goals with the new `DataProvider` component:

- it must run a (given) function to get the data
- it must expose the data (along with loading state and error messages) in a default slot so that any new component using the `DataProvider` can use those.


```vue
<script>
export default {
    props: {
        run: { type: Function, required: true }
    },
    data: () => ({
        data: null,
        error: null,
        loading: true
    }),
    async mounted() {
        try {
            const d = await this.$props.run();
            this.data = d.data;
        } catch (e) {
            this.error = e.message || 'Something went wrong';
        }
        this.loading = false;
    },
    render() {
        return this.$scopedSlots.default({
            data: this.data,
            error: this.error,
            loading: this.loading
        })
    }
}
</script>
```

Here's what is happening in that code:

- the component takes a prop called `run` which is defined as a function. For safety (and guarantee), we are also saying that this prop is `required` via `required: true`.
- the component initializes three local states: data, loading and error. By default, loading is set to true.
- the component has to run the given `run` function. So, we do this in the `mounted` hook. In this example, I am using the `async/await` syntax with a `try/catch` block to handle errors. The `mounted` hook runs the `run` function, gets the result (or error) and puts it in `data` (or `error`).
- finally, it exposes `data`, `error`, and `loading` properties to the default slot via the `render()` function.

For our use-case, the `run` function given as a prop is supposed to be an async function that returns some value or error.

Let's now create the `UserCard` component using the `DataProvider`. 

Old `UserCard` looks like this:

```vue
<template>
    <div class="card" v-if="loading">Loading...</div>
    <div class="card" v-else-if="data">
        <div class="avatar">
            <img :src="data.results[0].picture.medium" />
        </div>
        <div class="user-info">
            <div class="name">
                {{ data.results[0].name.first }}
                {{ data.results[0].name.last }}
            </div>
            <div class="email">{{ data.results[0].email }}</div>
            <div class="registered">
                Member since
                {{
                new Intl.DateTimeFormat("en-GB", {
                day: "numeric",
                month: "short",
                year: "numeric",
                }).format(new Date(data.results[0].registered))
                }}
            </div>
        </div>
    </div>
    <div v-else-if="error" class="card">Something went wrong.</div>
</template>
<script>
import axios from "axios";
export default {
    data: () => ({
        data: null,
        loading: true,
        error: null
    }),
    mounted() {
        this.loading = true;
        this.error = null;
        axios
            .get("https://randomuser.me/api/1.0/?seed=foobar")
            .then(r => {
                this.data = r.data;
                this.loading = false;
            })
            .catch(e => {
                this.error = e.response;
                this.loading = false;
            });
    }
};
</script>
```

Here's the new one:

```vue
<template>
    <DataProvider :run="getUser">
        <div class="card" slot-scope="{ data, user, loading }">
            <template v-if="loading">Loading...</template>
            <template v-else-if="error">Something went wrong</template>
            <template v-else-if="data">
                <div class="avatar">
                    <img :src="data.results[0].picture.medium" />
                </div>
                <div class="user-info">
                    <div class="name">
                        {{ data.results[0].name.first }}
                        {{ data.results[0].name.last }}
                    </div>
                    <div class="email">{{ data.results[0].email }}</div>
                    <div class="registered">
                        Member since
                        {{
                        new Intl.DateTimeFormat("en-GB", {
                        day: "numeric",
                        month: "short",
                        year: "numeric",
                        }).format(new Date(data.results[0].registered))
                        }}
                    </div>
                </div>
            </template>
        </div>
    </DataProvider>
</template>
<script>
import axios from "axios";
export default {
    components: {
        DataProvider: () => import('./DataProvider.vue')
    },
    data: () => ({
        run: axios.get('https://randomuser.me/api/1.0/?seed=foobar')
    })
};
</script>
```

Not a lot has changed in the `<template>` block other than the fact that now we're using a single wrapping `<div>` that uses the `slot-scope`.

```vue
<div slot-scope="{ data, loading, error }"> ... </div>
```

This is equivalent to writing:

```vue
<div slot-scope="object"> ... </div>
```

and then using `object.data`, `object.error` and `object.loading`.

We have simply used [object destructuring][objdestructuring] to get the properties directly.


Notice how the `<script>` block has shrunk in size.

All we do is import the `DataProvider` component, and then initialize the `run` property (as a function) so that we can pass it to `DataProvider`.

And just like that, we've outsourced the whole ( show loading -> try to get data -> show data if success / show error if error ) to another component that is generic enough that it can be used a million times and more now.

## Renderless to Render Component: Named Slots With Props

While we did reduce the code required for the times you create new components, there is something to be said about this part in the `UserCard` component:

```vue
<template v-if="loading">Loading...</template>
<template v-else-if="error">Something went wrong</template>
<template v-else-if="data">
```

Seems like this is something you'll be writing in every component in order to conditionally render the loading, error or data states.

That's repetition too and we can try and get rid of that.

To do this, we will go back to using the `<template>` block in our `DataProvider` and use something called "named slots".

A named slot is nothing more than a simple `<slot />` but with a specific name.

For example, let's say there's a component (called `Post`) like this:

```vue
<template>
    <div>
        <h1><slot name="heading" /></h1>
        <h3><slot name="summary" /><h3>
        <slot />
    </div>
</template>
```

We can use this component this way:

```vue
<template>
    <Post>
        <template v-slot:heading>Title of the post goes here</template>
        <template v-slot:summary>Post summary goes here</template>
        <!-- rest of the content -->
        ...
    </Post>
</template>
```

If we say `<slot />`, it's the default slot. This is similar to saying `<slot name="default" />`. (That's why we had `this.$scopedSlots.default()` back in the previous section).

But we can also give a name to each slot we create and then in the consuming component mention what content goes into which slot by using the `<template v-slot:slotName>` syntax.

Interestingly, we can also bind properties/data to each slot that we create.

For example:

```vue
<slot name="heading" :prefix="headingPrefix" :suffix="headingSuffix" />
```

Let's assume `headingPrefix` and `headingSuffix` are just strings that exist in the state of that component (ie, in data or computed property).

We can use this named slot like this:

```vue
<template>
    <template v-slot:heading="{ prefix, suffix }">
        {{ prefix }} Title of the Post Goes Here {{ suffix }}
    </template>
</template>
```

And the `prefix` and `suffix` strings will be rendered.

Notice three things here:
- we are simply using `<slot>` like a component
- we are giving it a name so that when actually using the component, we have the `slotName` for our `<template v-slot:slotName>` block
- and we are simply attaching props to the `<slot>` like we normally do to our components. This prop (in this case `prefix` and `suffix` become available for us later when we compose/create the new components)

The most interesting part for us, however, is that we can attach a `v-if` to these `<slot>`s.

So the plan to reduce/remove the overhead of writing `<template v-if="loading">` and so on for `data` and `error` is gone if we simply transfer that to the `DataProvider`. 

Here's our new `DataProvider` now:

```vue
<template>
    <div>
        <slot name="loading" :loading="loading" v-if="loading" />
        <slot name="error" :error="error" v-if="!loading && error" />
        <slot name="data" :data="data" v-if="!loading && data" />
    </div>
</template>
<script>
export default {
    props: {
        run: { type: Function, required: true }
    },
    data: () => ({
        data: null,
        error: null,
        loading: true
    }),
    async mounted() {
        try {
            const d = await this.$props.run();
            this.data = d.data;
        } catch (e) {
            this.error = e.message;
        }
        this.loading = false;
    }
};
</script>
```

We made these changes:

- we removed the `render()` function since we're going back to `<template>` blocks
- we added three `<slots>`, one each for `data`, `loading` and `error`
- and we also added a conditional `v-if` on each of these slots. They will only render/be accessible when the conditions are met.

Just like how we exposed `loading`, `data` and `error` states, we also expose them via the individual `<slot>`s. 

We can now refine our `UserCard` to be like this: (ignoring the `<script>` block because it remains the same)

```vue
<template>
    <DataProvider :run="run">
        <template v-slot:loading>Loading</template>
        <template v-slot:error="{ error }">
            {{ JSON.stringify(error) }}
        </template>
        <template v-slot:data="{ data }">
            <div class="card">
                <div class="avatar">
                    <img :src="data.results[0].picture.medium" />
                </div>
                <div class="user-info">
                    <div class="name">
                        {{ data.results[0].name.first }}
                        {{ data.results[0].name.last }}
                    </div>
                    <div class="email">{{ data.results[0].email }}</div>
                    <div class="registered">
                        Member since
                        {{
                        new Intl.DateTimeFormat("en-GB", {
                        day: "numeric",
                        month: "short",
                        year: "numeric",
                        }).format(new Date(data.results[0].registered))
                        }}
                    </div>
                </div>
            </div>
        </template>
    </DataProvider>
</template>
```

As you can see, we get rid of the `v-if`s in the code of `UseCard` and simply replace them with the appropriate slots. The `DataProvider` will take care of which slot to show/hide.

## Going Deeper with Global Source (via Store)

In an application which mostly stores and derives (computes) the data from a global store (like Vuex), the pattern we just devised may fall short.

For example, in most applications, a page-like component may dispatch an action via the store, which then updates a global store variable, which then computes the data for the page-like component which is then passed on to the children.

How do we handle loading, error and data in these cases?

Here's a simple Vuex store that gets and stores user information:

```vue
// ... all imports 

export default Vuex.Store({
    state: {
        user: {
            firstName: '',
            lastName: '',
            email: '',
            created: 0
        },
        fetchingUserInfo: false,
        fetchUserInfoError: null
    },
    mutations: {
        fetchUserInfoStart(state) { state.fetchingUserInfo = true },
        fetchUserInfoSuccess(state, payload) { state = ({
            ...state,
            user: ...payload
        }) },
        fetchUserInfoError(state, error) { state.fetchUserInfoError = error },
        fetchUserInfoEnd(state) { state.fetchingUserInfo = false }
    },
    actions: {
        async getUserInfo(ctx, payload) {
            ctx.commit('fetchUserInfoStart')
            try {
                const res = await axios.get(`https://randomuser.me/api/1.0/?seed=${payload.id}`)
                ctx.commit('fetchUserInfoSuccess', {
                    firstName: res.results[0].name.first,
                    lastName: res.results[0].name.last,
                    email: res.results[0].email,
                    created: res.results[0].registered
                })
            }
            catch (e) {
                ctx.commit('fetchUserInfoError', e)
            }
            ctx.commit('fetchUserInfoEnd')
        }
    }
})
```

Some things about this configuration:

- the core of the state is the `user` object. 
- the state also has keys for us to store the `loading` state of the request and the error object (if something goes wrong during the requesting phase)
- and we have the actions and the mutations that update the state based on what state the requesting phase is in: if it just started, we have `fetchUserInfoStart` which makes `fetchingUserInfo` true (ie, the loading state), if the request succeeded we have `fetchUserInfoSuccess` with the payload and so on.

So far, seems pretty normal.

How do we use this in our `DataProvider` abstraction?

While we could make the `UserInfo` component pass the `getUserInfo` action directly to the `DataProvider` but that doesn't solve really because how does `DataProvider` know which key of the store to look for when computing `data`, `error` and `loading` values?

We could pass the keys to look for but that would make `DataProvider` really dependent on a lot of props being passed down to it.

So how do we solve this?

### Data Structure In Stores

Did you notice that in the `state` of the Vuex Store, we have almost the same kind of tri-state that we have in new components? There's a data state (which is the `user` object), then there is the loading state (`fetchingUserInfo`) and finally the error state (`fetchUserInfoError`).

Seems like we could club these together into a single object and then somehow get `DataProvider` to read just that single object to get the state of the request!

Let's do it:

```vue
import Vue from "vue";
import Vuex from "vuex";
import axios from "axios";

Vue.use(Vuex);

export default new Vuex.Store({
  state: {
    user: {
      loading: false,
      error: null,
      data: null,
    },
  },
  mutations: {
    fetchUserInfoStart(state) {
      state.user.loading = true;
    },
    fetchUserInfoSuccess(state, payload) {
      Vue.set(state, "user", {
        ...state.user,
        data: payload,
      });
    },
    fetchUserInfoError(state, error) {
      state.user.error = error;
    },
    fetchUserInfoEnd(state) {
      state.user.loading = false;
    },
  },
  actions: {
    async getUserInfo(ctx, payload) {
      ctx.commit("fetchUserInfoStart");
      try {
        const res = await axios.get(
          `https://randomuser.me/api/1.0/?seed=${payload.id}`
        );
        ctx.commit("fetchUserInfoSuccess", {
          firstName: res.data.results[0].name.first,
          lastName: res.data.results[0].name.last,
          email: res.data.results[0].email,
          created: res.data.results[0].registered,
          avatar: res.data.results[0].picture.medium,
        });
      } catch (e) {
        ctx.commit("fetchUserInfoError", e);
      }
      ctx.commit("fetchUserInfoEnd");
    },
  },
});

```

We've done two things:

- modified the data structure to have a single key (`user`) that holds all the information about the request state
- modified mutations to match the data structure of our `user` object

Note that since we set `data: null` when we initialize the user, we use `Vue.set` when we modify the `user` object after a successful dispatch. This may not be necessary if we define `data: { ... }` even as we initialize it in `state` but this method comes in handy later.

The `getUserInfo` action remains the same.

Now, `DataProvider` can actually refer to the `user` key in the global store to compute the required states.

However, it would be the `UserInfo` card that hands the store key to the `DataProvider` component.

Here's how the `UserInfo` card looks:

```vue
<template>
    <DataProvider :run="getUserInfo" store="user">
        <template v-slot:loading>Loading...</template>
        <template v-slot:error="{ error }">Something went wrong: {{ error }}</template>
        <template v-slot:data="{ data }">
            <div class="card">
                <div class="avatar">
                    <img :src="data.avatar" />
                </div>
                <div class="user-info">
                    <div class="name">
                        {{ data.firstName }}
                        {{ data.lastName }}
                    </div>
                    <div class="email">{{ data.email }}</div>
                    <div class="registered">
                        Member since
                        {{
                        new Intl.DateTimeFormat("en-GB", {
                        day: "numeric",
                        month: "short",
                        year: "numeric",
                        }).format(new Date(data.created))
                        }}
                    </div>
                </div>
            </div>
        </template>
    </DataProvider>
</template>
<script>
export default {
    components: {
        DataProvider: () => import("./DataProvider")
    },
    methods: {
        getUserInfo() {
            return this.$store.dispatch("getUserInfo", { id: "foobar" });
        }
    }
};
</script>
```

In this, we are:
- sending the actual dispatch call to `DataProvider` and
- we are also telling that component which key of the store state to look into (`store="user"`)

Let's now make changes to the `DataProvider` component.

Remember that we have to do two things:

1. Instead of getting a value from the `run` function, we just run that function as it is simply a dispatch.
2. Instead of manually fiddling around with the `data`, `loading` and `error` keys, we just compute them from the store.

```vue
<template>
    <div>
        <slot name="loading" :loading="loading" v-if="loading" />
        <slot name="error" :error="error" v-if="!loading && error" />
        <slot name="data" :data="data" v-if="!loading && data" />
    </div>
</template>
<script>
export default {
    props: {
        run: { type: Function, required: true },
        store: { type: String, required: true }
    },
    computed: {
        loading() {
            return (
                this.$store.state[this.store] &&
                this.$store.state[this.store].loading
            );
        },
        data() {
            return (
                this.$store.state[this.store] &&
                this.$store.state[this.store].data
            );
        },
        error() {
            return (
                this.$store.state[this.store] &&
                this.$store.state[this.store].error
            );
        }
    },
    async mounted() {
        this.run();
    }
};
</script>
```

For safety reasons, in each of the computed property, we first make sure that our store indeed has the key that we're trying to access before computing the `.data`, `.loading` and `.error` properties of that key.

In our `UserInfo` component, specifically in the `<template v-slot:data>` block, we assume that the `data` has certain keys. But this may not be true always. For example, the server may respond with a completely different schema and our store might end up with an object that looks like:

```
data = {
    firstName: undefined,
    lastName: undefined,
    ... // everything else: undefined
}
```

Also, sometimes the data may be null as the response may have nothing in it even though the dispatch action finished successfully.

These kinds of cases are disastrous if you don't put in place sanity checks at every data handover junction. This can be done in a variety of ways and mostly depends on how you/your team decides to validate data coming into the system. I will leave that out as it is beyond the scope of this book.

[objdestructuring]: https://wesbos.com/destructuring-objects
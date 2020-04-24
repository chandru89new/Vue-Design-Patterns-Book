# Design Pattern #1: Creating Components Made Easy

## Introduction

Most of your time as a Vue.js developer is spent creating (or extending) components. If you work on a project that just got started, you'll most likely be creating new components all day long.

Some components are simple, some are sophisticated beasts. But most components – no matter how simple or sophisticated – have a small part of shared logic: data states and one data-getter function that runs on the `mounted()` or `created()` hooks.

If we take a component that renders a UI based on data, we automatically have three states for this component:

- a loading state to show while the data is fetched
- an error state in case there was some issue with the data fetching
- a data state (which can be split into two: no data and data)

In the wild, a component typically looks like this:

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

## Use Case / Example

Let's take the case of this simple user-info card. It shows an avatar, a name and some other information of a specific user. We get this user by making an API call (in this case, to `https://randomuser.me/api/1.0/?seed=foobaz`). (Just to ensure you understand the properties used in the upcoming code, [look at the response of that call here](https://randomuser.me/api/1.0/?seed=foobaz).)

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

Did you notice that there are some things that are quite common here?

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
        return this.$scopedSlots({
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

Not a lot has changed in the `<template>` block other than the fact that now we're using one single wrapping `<div>` that uses the `slot-scope`.

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

## Renderless to Render Component: Custom Slots

## Going Deeper with Global Source (via Vuex)
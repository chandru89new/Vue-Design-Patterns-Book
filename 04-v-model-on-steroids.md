# Design Pattern #3: Unleashing The True Potential Of v-model In Complicated Forms And Other Places

## Introduction

Most people use `v-model` in Vue to attach/bind state to a native form element like `input` or `select`. 

But `v-model` is designed to help in more than form elements. All custom components you create can use `v-model`. But why would one use that? How can that help?

That is what we'll look at in this pattern.

## Two-way Communication: Child Components, Props and Event Emitters

In Vue, like in React, data-flow is mostly one-way. Parent components hand over data to children and if a child component wants to edit that data (which is a prop), it emits an event up to the parent component. The parent component then handles the event and mutates the data being sent to the children.

This pattern is enforced by Vue so if your child component tries to modify the `prop` it receives, you will see warnings/errors in your dev console.

However, you do notice that this does not happen when you use `v-model`. This is because `v-model` is sort of a combination of `v-bind` (shortened as `:`) and `v-on` (shortened as `@`). `v-model` enables two-way data binding in Vue.

For example, these two pieces of code mean the same:

```vue
<input v-model="firstName" />
```

```vue
<input :value="firstName" @input="e => firstName = e.target.value" />
```

When a `v-model` is used, the value is bound to the component and it is updated when the value is modified by the component.

We can use `v-model` to reduce code across components.


Let's take two examples.

The first one is a simple counter. This is a contrived example, so instead of having a simple one-component counter, we'll have a two-component counter.

The child component is the one that shows the count, the increment and decrement buttons.

The parent component simply uses the child component but it provides the count as a prop to the child.

```vue
<!-- Parent -->
<template>
    <ChildComponent :count="count" @increment="count = count + 1" @decrement="count = count - 1" />
</template>
<script>
export default {
    components: {
        ChildComponent: () => import('./ChildComponent')
    },
    data: () => ({
        count: 10
    })
}
</script>
```

```vue
<!-- Child -->
<template>
    <div>
        <button @click="dec"> - </button>
        <span>{{ count }}</span>
        <button @click="inc"> + </button>
    </div>
</template>
<script>
export default {
    props: ['count'],
    methods: {
        inc() {
            this.$emit('increment')
        },
        dec() {
            this.$emit('decrement')
        }
    }
}
</script>
```

As you can see, the child component emits two events (`increment` and `decrement`) which are then handled by the parent component. This is a typical pattern you'll find in most Vue applications.

However, we can reduce the overhead in the parent and child component by using `v-model`.

Here's how the parent looks:

```vue
<!-- Parent -->
<template>
    <ChildComponent v-model="count" />
</template>
<script>
export default {
    components: {
        ChildComponent: () => import('./ChildComponent')
    },
    data: () => ({
        count: 10
    })
}
</script>
```

Notice that we've removed the `v-bind:count` (shortened as `:count`) and the event listeners `@increment` and `@decrement`, and replaced them with a simple `v-model="count"`.

Now, let's modify the child component:

```vue
<!-- Child -->
<template>
    <div>
        <button @click="dec"> - </button>
        <span>{{ count }}</span>
        <button @click="inc"> + </button>
    </div>
</template>
<script>
export default {
    props: ['count'],
    model: {
        prop: 'count',
        event: 'change'
    },
    methods: {
        inc() {
            const newCount = this.count + 1
            this.$emit('change', newCount)
        },
        dec() {
            const newCount = this.count - 1
            this.$emit('change', newCount)
        }
    }
}
</script>
```

Here's what we've changed in the child component:

1. Firstly, we have introduced a new `model` object in the options:

```js
model: {
    prop: 'count',
    event: 'change'
}
```

`model.prop` says which prop value is to be considered to the model. This comes in handy when the child component emits a specific event with a specific value and Vue automatically updates the `prop` (refered to in the `model.prop`) with the specific value.

`model.event` says which event to listen for. In our case, it's called `change`.

2. Next, we have updated the `inc()` and `dec()` methods to compute a new value based on the current value and then emit an event (`this.$emit('change')`) with the new/updated value. As you can see, the name of the event we emit is the same as `model.event`. This is required so that Vue knows which event to listen for in order to update the `prop` from the child.

And just like that, you can allow child components to edit the prop it receives.

Note that you can only edit one prop through a `v-model` binding.

Let's take another example, this time a little sophisticated where the user can actually modify values through form elements.

```vue
<!-- Parent -->
<template>
    <ChildComponent :details="authorDetails" @change="updateAuthorDetails" />
</template>
<script>
export default {
    components: {
        ChildComponent: () => import('./ChildComponent.vue')
    },
    data: () => ({
        authorDetails: {
            name: 'Jason Bourne',
            age: 45,
            occupation: 'Assassin'
        }
    }),
    methods: {
        updateAuthorDetails(d) {
            this.authorDetails = { ...d }
        }
    }
}
</script>
```

```vue
<!-- Child -->
<template>
    <div>
        <div><input v-model="name" /></div>
        <div><input v-model="age" /></div>
        <div><input v-model="occupation" /></div>
        <div>
            <button @click="save">Save</button>
        </div>
    </div>
</template>
<script>
export default {
    props: ['details'],
    data() {
        return {
            name: this.details.name,
            age: this.details.age,
            occupation: this.details.occupation
        }
    },
    methods: {
        save() {
            const data = { name: this.name, age: this.age, occupation: this.occupation }
            this.$emit('change', data)
            return
        }
    }
}
</script>
```

In this example, we're using a Parent component to send some author details down to a child component as prop. The child component has a `Save` button which, when clicked, emits a `change` event with the new/updated `name`, `age` and `occupation`. The parent "catches" this event and then modifies the data accordingly.

Let's first remove stuff from the Parent component.

```vue
<!-- Parent -->
<template>
    <ChildComponent v-model="authorDetails" />
</template>
<script>
export default {
    components: {
        ChildComponent: () => import('./ChildComponent.vue')
    },
    data: () => ({
        authorDetails: {
            name: 'Jason Bourne',
            age: 45,
            occupation: 'Assassin'
        }
    })
}
</script>
```

- We have removed the bindings and event handlers on the `<ChildComponent>` used in the Parent and replaced it with just `v-model`
- We have also removed the `methods` because we don't need that.

This means we've got to change some things in the child.

```vue
<!-- Child -->
<template>
    <div>
        <div><input v-model="name" /></div>
        <div><input v-model="age" /></div>
        <div><input v-model="occupation" /></div>
        <div>
            <button @click="save">Save</button>
        </div>
    </div>
</template>
<script>
export default {
    props: ['details'],
    model: {
        prop: 'details',
        event: 'change'
    },
    data() {
        return {
            name: this.details.name,
            age: this.details.age,
            occupation: this.details.occupation
        }
    },
    methods: {
        save() {
            const data = { name: this.name, age: this.age, occupation: this.occupation }
            this.$emit('change', data)
            return
        }
    }
}
</script>
```

- As in the previous example, we've simply added a small object called `model` which has two keys: `prop` and `event`.

You will notice that now, without any mutation method in the Parent, the prop being passed to the child from the parent automatically updates when you click on the `Save` button.

Why?

Because the `Save` button triggers the `change` event (which is the event mentioned in `model.event`) and the new data coming along with that event (`this.$emit('change', data)`) replaces the old data in `details` (which is mentioned in `model.prop`).


You will notice that even though we use `v-model`, we're not really getting rid of the initialization of the local state in the `data()` function in the child component.

We do this because the design of the child component is like that: we let the user edit all (or any) of the three fields and then save only when the user clicks on `Save`. That is, the `change` event has to fire only when user clicks on `Save` button.

However, we can also have a design where we remove the need for the `Save` button and instead, update the value of `authorDetails` (passed in the prop `details`) even as the user updates each field.

To do this, we have to remove `v-model` from the `<input>`s and emit the `keyup` event from the `<input />`s whenever the value is being mutated by the user. That is, we're simply splitting the `v-model` in the `<input>`s.

So, we have:

```vue
<!-- Child -->
<template>
    <div>
        <div><input :value="details.name" @keyup="e => save({ name: e.target.value })" /></div>
        <div><input :value="details.age" @keyup="e => save({ age: e.target.value })" /></div>
        <div><input :value="details.occupation" @keyup="e => save({ occupation: e.target.value })" /></div>
    </div>
</template>
<script>
export default {
    props: ['details'],
    model: {
        prop: 'details',
        event: 'change'
    },
    methods: {
        save(data) {
            const updatedData = {
                ...this.details,
                ...data
            }
            this.$emit('change', updatedData)
            return
        }
    }
}
</script>
```

## Letting Child Components Run the Roost

So far, we've only talked about contrived examples where the parent has the data hard-coded in it.

What if the data is coming from an API and when it is mutated, it has to send a PUT, PATCH, POST or DELETE request to the API and handle the response?

We can do that too while still sticking to `v-model` as our primary tool.

Here's one example of a simple application where:
- the parent gets a data about a user from some API source
- passes the data on to the child using `v-model`
- and when the data is updated on the child, it has to be saved in the backend through an API call

To mimic a promise-based backend, I have a script that uses localStorage as a backend and returns every response after a timeout of 1 second (1000ms).

Here's the basic setup of that script:

```js
// Storage.js

// global state
const KEY = "db";

// manually add some dummy data to localStorage so that this whole app works
if (!localStorage.getItem(KEY)) {
  localStorage.setItem(
    KEY,
    '[{"userId":1,"name":"Jason Bourne","age":45,"occupation":"Assassin"},{"userId":2,"name":"James Bond","age":54,"occupation":"MI6"}]'
  );
}

// loads data from localstorage and parses it. this is the data used by all the functions
const loadData = () => {
  return JSON.parse(localStorage.getItem(KEY) || "[]");
};

// saves the data in localstorage by first stringifying it
const saveData = (DATA) => {
  const d = JSON.stringify(DATA);
  localStorage.setItem(KEY, d);
  return true;
};

export default {
  get: (userId) => {
    return new Promise((res) => {
      setTimeout(() => {
        const DATA = loadData();
        res(DATA.find((d) => d.userId === userId));
      }, 1000);
    });
  },
  delete: (userId) => {
    return new Promise((res) => {
      setTimeout(() => {
        const DATA = loadData();
        const newData = DATA.filter((d) => d.userId !== userId);
        saveData(newData);
        res(true);
      }, 1000);
    });
  },
  patch: ({ userId, name, age, occupation }) => {
    return new Promise((res) => {
      setTimeout(() => {
        const DATA = loadData();
        const newDATA = DATA.map((d) =>
          d.userId === userId ? { ...d, name, age, occupation } : d
        );
        saveData(newDATA);
        res({ userId, name, age, occupation });
      }, 1000);
    });
  },
};
````

Let's see what the Parent looks like:

```vue
<!-- Parent -->
<template>
    <ChildComponent v-if="!loading" v-model="authorDetails" />
</template>
<script>
import Storage from "./Storage";
export default {
    components: {
        ChildComponent: () => import("./ChildComponent")
    },
    data: () => ({
        authorDetails: null,
        loading: true
    }),
    async mounted() {
        this.authorDetails = await Storage.get(1);
        this.loading = false;
    }
};
</script>
```

Here, we are calling `Storage.get(1)` to get the details of `userId` 1. That is being sent to the child component via `authorDetails`.

The child component looks like this:

```vue
<template>
    <div>
        <div>
            <input v-model="name" />
        </div>
        <div>
            <input v-model="age" />
        </div>
        <div>
            <select v-model="occupation">
                <option value="Assassin">Assassin</option>
                <option value="MI6">MI6</option>
                <option value="Author">Author</option>
            </select>
        </div>
        <div>
            <button>Update</button>
        </div>
    </div>
</template>
<script>
export default {
    model: {
        prop: "details",
        event: "update"
    },
    props: ["details"],
    data() {
        return {
            name: this.details.name,
            age: this.details.age,
            occupation: this.details.occupation,
            userId: this.details.userId
        };
    }
};
</script>
```

Here, we're just initializing some local state using the prop (`details`) and then showing a form with that local state filled in.

The user can edit things here (like `name`, `age`, `occupation`) and then apparently click on `Update` to update the data. This update functionality is not yet there.


Let's think about what update needs to do. It has to call `Storage.patch` with the new, updated information. Now, the question is: who calls `Storage.patch`?

If we were not using `v-model`, we'd be emitting an event with the updated information from the child to the parent component and the parent component would call `Storage.patch` with the updated information. 

Shipping `Storage.patch` to child does not seem right because the child component shouldn't be dependent on `Storage` as a dependency. (Notice that it does not get the data from `Storage.get` directly; only the parent sends it)

One thing we can do is pass a function to run as a prop _a la_ React.

Let's edit our parent component to create that function and hand it over to the child as a prop:

```vue
<!-- Parent -->
<template>
    <ChildComponent v-if="!loading" v-model="authorDetails" :update="updateDetails" />
</template>
<script>
import Storage from "./Storage";
export default {
    components: {
        ChildComponent: () => import("./ChildComponent")
    },
    data: () => ({
        authorDetails: null,
        loading: true
    }),
    async mounted() {
        this.authorDetails = await Storage.get(1);
        this.loading = false;
    },
    methods: {
        updateDetails(newDetails) {
            return Storage.patch(newDetails)
        }
    }
};
</script>
```

We create a new function `updateDetails` that takes the updated details as a param and then passes it on to `Storage.patch` and returns that function. (A promise).

And it sends this function as a prop to the child component as `:update="updateDetails"`.

Let's now add the `Update` functionality to our child component:

```vue
<template>
    <div>
        <div>
            <input v-model="name" />
        </div>
        <div>
            <input v-model="age" />
        </div>
        <div>
            <select v-model="occupation">
                <option value="Assassin">Assassin</option>
                <option value="MI6">MI6</option>
                <option value="Author">Author</option>
            </select>
        </div>
        <div>
            <button @click="updateData">Update</button>
        </div>
    </div>
</template>
<script>
export default {
    model: {
        prop: "details",
        event: "update"
    },
    props: ["details", "update"],
    data() {
        return {
            name: this.details.name,
            age: this.details.age,
            occupation: this.details.occupation,
            userId: this.details.userId
        };
    },
    methods: {
        updateData() {
            const d = {
                name: this.name,
                age: this.age,
                occupation: this.occupation,
                userId: this.userId
            };
            this.update(d).then(() => this.$emit("update", d));
        }
    }
};
</script>
```

Here:

- We add `@click="updateData"` to the `button` 
- And we add the `updateData` which simply uses the local state to create the new / updated details and then passes it on to the `update` prop (which is a function we received from the parent). After `this.update` runs, we emit the `update` event so that the model kicks in and updates the data everywhere (parent, child).

Notice that since `Storage.patch` returns a promise, the parent is actually passing down a function that returns a promise to the child.

And so, the child can take advantage of that to handle success and failure states. In our simple example, we're simply using `.then()` to handle the success state. You can handle the failures via `.catch()` chained to the `.then()`.

## Using with Store

In the examples above, we've tasked a parent component to directly talk to the API (in our case, the `Storage` module). However, in most applications, this job is delegated to the store (Vuex actions, typically). This way, the parent only dispatches actions and takes/computes the value from the store state.

In this case, since we use a `computed` property, we cannot mutate that directly from either child or the parent. Instead, we use computed setters.

```vue
<!-- Parent -->
<template>
  <ChildComponent v-model="authorDetails" :key="`${JSON.stringify(authorDetails)}`" />
</template>
<script>
export default {
  components: {
    ChildComponent: () => import("./ChildComponent")
  },
  mounted() {
    this.$store.dispatch("getUser", { userId: 1 });
  },
  computed: {
    authorDetails: {
      get() {
        return this.$store.getters.getUser;
      },
      set(v) {
        return this.$store.dispatch("updateUser", v);
      }
    }
  }
};
</script>
```

What we're doing is fairly simple:

- on component's `mounted` hook, we dispatch a store action that gets data from the backend for `userId=1`
- the actual data that is being passed to the child component as a v-model is `authorDetails` which is computed from the store. Note that instead of simply using this:

```vue
authorDetails() {
    return this.$store.getters.getUser
}
```

we are using:

```vue
authorDetails: {
  get() {
    return this.$store.getters.getUser;
  },
  set(v) {
    return this.$store.dispatch("updateUser", v);
  }
}
```

This is because if you are going to mutate any computed property, you need to use the `get()`-`set()` idiom.

What's happening in the code is that whenever the child component mutates data in `authorDetails` (by virtual of it being `v-model`), the parent component will call the function in `set(v)` of the `authorDetails` computed property. That function incidentally happens to be a dispatch action that will update the data in the backend and the store.

The child component has some minor changes:

```vue
<template>
  <div>
    <div>
      <input v-model="name" />
    </div>
    <div>
      <input v-model="age" />
    </div>
    <div>
      <select v-model="occupation">
        <option value="Assassin">Assassin</option>
        <option value="MI6">MI6</option>
        <option value="Author">Author</option>
      </select>
    </div>
    <div>
      <button @click="updateData">Update</button>
    </div>
  </div>
</template>
<script>
export default {
  model: {
    prop: "details",
    event: "update"
  },
  props: ["details"],
  data() {
    return {
      name: this.details.name,
      age: this.details.age,
      occupation: this.details.occupation,
      userId: this.details.userId
    };
  },
  methods: {
    updateData() {
      const d = {
        name: this.name,
        age: this.age,
        occupation: this.occupation,
        userId: this.userId
      };
      this.$emit("update", d);
    }
  }
};
</script>
```

We've simply removed two things: the prop `update` in the props list and in the `updateData()` method, we removed the line that called the `this.update()` function. Instead, we simply emit the `update` function to the parent (which the `v-model` takes care of).


One important thing to note here is the `:key` in `<ChildComponent ...`. We do this because when the parent component first loads, the `authorDetails` is actually empty but after `mounted()`, `authorDetails` is updated.

However, the child component does not update unless we ask it explicitly to. If it does not update, we see empty inputs in the child component even though `authorDetails` has subsequently been loaded.

In order to prevent this, we need to force the child component to load again when `authorDetails` is updated (ie, the store dispatch resolves and updates the store value). That is why we use a `:key`. Since the value of the `key` gets updated, that forces the child component to re-render with the new data.

If you do not want to use `key` to force a re-render with the updated data, you can attach a `watch` to `$props.details` in the child component and update the local state from there.

## But What About The Loading States?

In our examples, we haven't talked about the loading state.

As an example, when the parent gets data from the backend, it takes about a second to fetch the data and then hand it over to the child component. When that happens, how to handle the loading state?

The other loading state is when the child mutates the data which triggers the `updateUser` action in the parent. When this happens, who handles the loading state? The child? Or the parent?

Both are viable options. However, on first glance, you'll notice that it is easier for the parent to know the loading state for both the `getUser` and `updateUser` actions as those actions are promises returning a value.


Even better would be to use the Tuple technique introduced in Chapter 1 where the user details is stored as a tuple of `data`, `loading` and `error` fields; this way, the child will get the loading state automatically because the store action will mutate the `loading` key according to the state of the API response.
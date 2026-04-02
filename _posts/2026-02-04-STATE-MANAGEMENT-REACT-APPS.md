---
title: "The Golden Standard and the Evolution of State Management in React"
date: 2026-04-02 00:00:00 +0000
categories: [React, Architecture, State Management]
tags: [React, React Native, Redux, React Query, Intermediate, State Management]
description: "A deep dive into the transition from local and global state ceremonies to the modern Server State approach."
author: diogoizele
toc: true
image:
  path: /assets/img/004/cover.png
---

Since my first professional opportunity, I have always been concerned with understanding the **ideal way** to do something. I knew **what** needed to be done and I knew **how to do it** my way, but I wasn't certain if my "how" was the best way. I believe this is a common issue for those with some degree of perfectionism, that ghost that haunts us by suggesting there is a better way than ours to achieve the same result.

Among many concepts in frontend programming, state management has always been one of those that most sparked my interest in diving deeper to understand the ideal "how-to."
## Local State

Initially, I learned that I could handle any external API call by creating a function and importing `axios` (or one of its instances) directly into the screen (React component). As simple as:


```jsx
export const Products = () => {
  const [products, setProducts] = useState([]);

  const fetchProducts = async () => {
    const { data } = await axios.get("https://my-business/api/products");
    setProducts(data);
  };

  useEffect(() => {
    fetchProducts();
  }, []);

  return (
    <div>
      <ul>
        {products.map(product => (
          <li key={product.id}>{product.name}</li>
        ))}
      </ul>
    </div>
  );
};
```

This worked. However, as the screen grew, the number of local states resulting from API calls or conditional rendering also increased alongside improvements in user experience.

At one point, we had:

```jsx
const API_URL = "https://my-business/api";

export const Products = () => {
  const [products, setProducts] = useState([]);
  const [isLoading, setIsLoading] = useState(true);
  const [fetchProductError, setFetchProductError] = useState(null);
  const [editProductError, setEditProductError] = useState(null);
  const [deleteProductError, setDeleteProductError] = useState(null);
  const [productName, setProductName] = useState("");
  const [productPrice, setProductPrice] = useState(0);

  const fetchProducts = async () => {
    setIsLoading(true);
    try {
      const { data } = await axios.get(`${API_URL}/products`);
      setProducts(data);
    } catch (error) {
      setFetchProductError(error?.response?.data?.message || "Error fetching products");
    } finally {
      setIsLoading(false);
    }
  };

  const handleEditProduct = async (id) => {
    setIsLoading(true);
    try {
      await axios.put(`${API_URL}/products/${id}`, {
        name: productName,
        price: productPrice
      });
      await fetchProducts();
      Toast.show({ type: "success", message: "Product edited successfully" });
      setProductName("");
      setProductPrice(0);
    } catch (error) {
      setEditProductError(error?.response?.data?.message || "Error editing product");
    } finally {
      setIsLoading(false);
    }
  };

  const handleDeleteProduct = async (id) => {
    setIsLoading(true);
    try {
      await axios.delete(`${API_URL}/products/${id}`);
      await fetchProducts();
      Toast.show({ type: "success", message: "Product removed successfully" });
    } catch (error) {
      setDeleteProductError(error?.response?.data?.message || "Error removing product");
    } finally {
      setIsLoading(false);
    }
  };

  ...
  
  return <div>...</div>;
};
```

Some parts of the code above were omitted, such as effect hooks and JSX, but it is already clear that in a simple screen containing only part of the CRUD operations, we need to create and manage numerous local states. During this phase, I remember a colleague recommending I read the documentation and usage examples for `useReducer`.

Using this hook, even in a simple example like ours, can give the impression that we are organizing the house. By defragmenting various states into a single object and using the _action/dispatch_ pattern to update them, the structure truly seems to improve:

```jsx
const API_URL = "https://my-business/api";

// Products.reducer.js

export const initialState = {
  products: [],
  isLoading: true,
  fetchProductError: null,
  editProductError: null,
  deleteProductError: null,
  form: {
    productName: "",
    productPrice: 0
  }
};

export function reducer(state, action) {
  switch (action.type) {
    case "products_request":
      return {
        ...state,
        products: [],
        isLoading: true,
        fetchProductError: null
      };
    case "products_success":
      return {
        ...state,
        products: action.payload,
        isLoading: false
      };
    case "products_failure":
      return {
        ...state,
        isLoading: false,
        fetchProductError: action.error
      };
    case "set_loading":
      return {
        ...state,
        isLoading: action.payload
      };
    case "edit_error":
      return {
        ...state,
        editProductError: action.error
      };
    case "delete_error":
      return {
        ...state,
        deleteProductError: action.error
      };
    case "set_form_data":
      return {
        ...state,
        form: {
          ...state.form,
          [action.payload.name]: action.payload.value
        }
      };
    case "reset_form_data":
      return {
        ...state,
        form: { ...initialState.form }
      };
    default:
      return state;
  }
}

// Products.js

export const Products = () => {
  const [state, dispatch] = useReducer(reducer, initialState);

  const fetchProducts = async () => {
    dispatch({ type: "products_request" });
    try {
      const { data } = await axios.get(`${API_URL}/products`);
      dispatch({ type: "products_success", payload: data });
    } catch (error) {
      dispatch({
        type: "products_failure",
        error: error?.response?.data?.message || "Error fetching products"
      });
    }
  };

  const handleEditProduct = async (id) => {
    dispatch({ type: "set_loading", payload: true });

    try {
      await axios.put(`${API_URL}/products/${id}`, {
        name: state.form.productName,
        price: state.form.productPrice
      });

      await fetchProducts();
      Toast.show({ type: "success", message: "Product edited successfully" });
      dispatch({ type: "reset_form_data" });
    } catch (error) {
      dispatch({
        type: "edit_error",
        error: error?.response?.data?.message || "Error editing product"
      });
    } finally {
      dispatch({ type: "set_loading", payload: false });
    }
  };

  const handleDeleteProduct = async (id) => {
    dispatch({ type: "set_loading", payload: true });

    try {
      await axios.delete(`${API_URL}/products/${id}`);
      await fetchProducts();
      Toast.show({ type: "success", message: "Product removed successfully" });
    } catch (error) {
      dispatch({
        type: "delete_error",
        error: error?.response?.data?.message || "Error removing product"
      });
    } finally {
      dispatch({ type: "set_loading", payload: false });
    }
  };

  const handleInputChange = (event) => {
    const { value, name } = event.target;
    dispatch({ type: "set_form_data", payload: { value, name } });
  };

  ...

  return (
    <div>
      ...
      <form>
        <input
          name="productName"
          onChange={handleInputChange}
          value={state.form.productName}
        />
      </form>
    </div>
  );
};
```

In the example above, some parts were also omitted, but it is sufficient to demonstrate the use of the `useReducer` hook instead of multiple `useState` calls. Here, we have a defined initial state and a set of well-typed actions, making state transitions more predictable. Furthermore, an `action` can update multiple state values simultaneously, as seen in the `products_*` actions.

Despite this, I never saw much practical advantage in this pattern for cases like this. At first glance, it seems we are making state management more organized and independent. Indeed, we gain a pure and deterministic function (`reducer`) that doesn't depend on the React lifecycle, along with an `initialState` decoupled from the UI. This is positive.

However, we still end up with a ["god state"](https://en.wikipedia.org/wiki/God_object), which is small for now but tends to grow and concentrate everything that happens on that screen. Is this inherently bad? It depends. In our example, no. The state is relatively small, and there is no real penalty in keeping it this way if size is the only criterion. The point is that when we start talking about coupling, granularity, and maintainability, this type of grouping begins to stand out. It causes no problems now, but it could hinder scalability later on.

## The Promise of a Modular Design

As I progressed, I gradually learned that the visual interface should not make API calls or talk directly to external layers. It should only collect events and render states. From this, I realized that more mature architectures follow a flow where the UI consumes a global state, while this global layer possesses a communication mechanism with the external world. Whether through a [generator function](https://developer.mozilla.org/en-US/docs/Web/js/Reference/Statements/function*) in [Redux-Saga](https://redux-saga.js.org/docs/introduction/GettingStarted) or an _[async thunk](https://en.wikipedia.org/wiki/Thunk)_ in [RTK](https://redux-toolkit.js.org/introduction/getting-started), this layer waits for the API response and handles `REQUEST`, `SUCCESS`, and `FAILURE` states.

This model always seemed much more mature and ideal to me, precisely because it separates responsibilities between layers. If tomorrow we stop using an Axios client and migrate to Firebase, we only need to adjust the Saga or Thunk to meet the new contract, and all screens continue working without any changes.

It seemed perfect. It seemed like the ideal design. That’s what I believed for a long time.

![Scheme](assets/img/004/scheme.svg)

In the example above, the UI triggers an action, very similar to the `dispatch` of `useReducer` (the [mechanism](https://www.freecodecamp.org/news/an-introduction-to-the-flux-architectural-pattern-674ea74775c9/) used in the hook and in Redux is essentially the same). This action is observed or directly handled by a **side-effect middleware**, a layer responsible for communicating with the API or any external service. Eventually, this layer receives a response, success or error, and processes it into a reducer or global state, which then returns the data for the UI to react and re-render.

I won’t go into implementation details with the libraries, but the important thing here is to understand that this pattern represents the "Gold Standard" of the era when Redux dominated the market. Most projects I worked on, created around 2018-2021, followed exactly this type of architecture.

As I said, until recently, I myself looked at this model and still considered it ideal and modern. It was in this way, in fact, that I developed the [DinHero](https://github.com/diogoizele/Din.hero) app, used as a laboratory in my portfolio. In it, I created a layer that interacts with the UI called a _store_ using Redux Toolkit, and through _thunks_, I bridge the gap to my _services_, which at the moment are an abstraction for communication with Firebase.

The problem is that this approach adds a layer of [boilerplate](https://en.wikipedia.org/wiki/Boilerplate_code) with very limited practical utility. It quickly transforms more into ceremony than something that truly delivers value. In practice, you rarely use the "G" in "**global state**": many actions are triggered and consumed in only a single point of the application, something that could be perfectly resolved with **local state**.

This line of thought led me to question the real utility of this architecture and also the indiscriminate use of global state management libraries. If the state is **not** consumed globally (i.e., in multiple places in the app), does it still make sense to maintain a tool whose central purpose is precisely that?

What now? Does this mean patterns have regressed and we should go back to calling the API directly on the screen using only local states? As controversial as it may seem, the short answer is: yes. But not in the naive way I showed at the beginning of the article.

The truth is that modern React application architecture is **anemic in global state**. Most data handled by an application is not "client state," but **[server state](https://dev.to/jeetvora331/server-state-vs-client-state-in-react-for-beginners-3pl6)**: data fetched from a server that needs to exist only temporarily in memory, subject to synchronization, invalidation, and refetching. The type of data for which Redux was never truly the ideal tool.

## Server State in the Frontend?

While deepening my studies on state management in modern architectures, I encountered the notion of server state. In the frontend context, the term isn't as widespread, but the idea is straightforward: it refers to data obtained from an external API that remains temporarily in local memory, equivalent to being stored in a `useState`.

The tool that consolidated this model is **React Query**. Emerging around 2020, it replaces the inappropriate use of global state solutions employed just to store server responses, something that is not global by nature.

With React Query, external service calls are made through hooks that expose loading indicators, error states, success or failure callbacks, and a complete caching system with invalidation and refetching. This eliminates the need for extensive ceremonial layers created solely to separate responsibilities that are already resolved natively by the tool itself.

Rewriting our multiple local state example for a React Query-based approach, we get:

```jsx
const API_URL = "https://my-business/api/"

export const Products = () => {
  const queryClient = useQueryClient()
  const [productName, setProductName] = useState("")
  const [productPrice, setProductPrice] = useState(0)

  const {
    data: products = [],
    isLoading,
    error: fetchProductError,
  } = useQuery({
    queryKey: ["products"],
    queryFn: async () => {
      const { data } = await axios.get(`${API_URL}/products`)
      return data
    },
  })

  const editProduct = useMutation({
    mutationFn: async (id) =>
      axios.put(`${API_URL}/products/${id}`, {
        name: productName,
        price: productPrice,
      }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["products"] })
      Toast.show({ type: "success", message: "Product edited successfully" })
      setProductName("")
      setProductPrice(0)
    },
  })

  const deleteProduct = useMutation({
    mutationFn: async (id) => axios.delete(`${API_URL}/products/${id}`),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["products"] })
      Toast.show({ type: "success", message: "Product removed successfully" })
    },
  })

  ...

  return (
    <div>
      ...
    </div>
  )
}
```

Notice that in this model, the API call happens directly in the UI layer again. The question is whether this should be done. The answer remains: it depends. In many cases, it is worth encapsulating the external boundary in a pure service, keeping the UI responsible only for orchestrating already prepared data. This type of abstraction centralizes transformation rules and leaves the interface free of infrastructure details.

```ts
export const ProductService = {
  getAll: async (): Promise<Product[]> => {
    const { data } = await apiClient.get<ProductResponse[]>(`/products`)

    const mappedProducts = data.map(product => ({
      id: product.id,
      name: product.description,
      category: product.product_category,
    }))

    return mappedProducts
  }
}
```

In this way, the chain is explicit: the UI calls a hook like `useProducts`, the hook triggers React Query, and React Query executes `ProductService.getAll`. This chaining organizes responsibilities, but it doesn't mean every intermediate layer needs to exist in every scenario.

If you look at this from a pragmatic point of view, the question arises naturally: when the API already returns exactly the necessary format and no transformation is required, does adding a function that merely passes the Axios call along generate any concrete benefit? This is, in fact, an empty abstraction, structurally correct, but delivering no practical value.

```TypeScript
export const ProductService = {
  getAll: async (): Promise<Product[]> => {
    return await apiClient.get<ProductResponse[]>(`/products`)
  }
}
```

The layer exists primarily to isolate the external boundary and apply transformations, not to "hide Axios." Creating an entire file just to delegate a call is bureaucracy; real utility appears when the service fulfills a concrete function:

- Transforming the response
- Normalizing types
- Centralizing endpoint origins
- Allowing reuse across multiple screens
- Justifying direct unit tests

The service acts as an adapter: the wall socket (the API) can have any shape, but my device (the app) only operates with a specific plug standard.

## Final Thoughts

At the end of the day, the search for the "ideal how-to" leads to a simple realization: **architecture is only good when it solves the problem without becoming part of it**. For years, React applications carried excessive ceremony around state, especially in the era when any data coming from the server was automatically treated as global. Layers were created just to justify the tool itself, not to serve the application.

This trajectory highlights that "state" was never a single block, but a set of different responsibilities that, for convenience, we treated as if they were the same. First, we concentrated everything in the UI, then we grouped it, then we externalized it, then we sophisticated it. Ultimately, the discipline of state management is the art of separating what belongs to the interface, what belongs to the external world, and what belongs to the domain.

By adopting the concept of **Server State** and tools like React Query, we are not returning to local state; we are simply resizing global state to its legitimate role. It continues to exist, but only where it makes sense. The rest returns to the natural flow: server data lives as server data, with its own cycle, its own cache, and its own rules. It is an adjustment that reduces friction, eliminates artificial layers, and restores clarity to the project.
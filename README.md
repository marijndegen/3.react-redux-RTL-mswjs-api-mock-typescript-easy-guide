# react-redux-RTL-mswjs-api-mock-typescript-easy-guide

This guide assumes create react app has been succesfully done, so follow this guide before getting into this one: https://github.com/marijndegen/redux-create-react-app-easy-guide

We will create a real api with nodejs (written in javascript) and adjust the react (typescript) so it consumes the real api. The tests will mock the real api using the mock service worker (MSW), so regardless of the http library (fetch, axios etc.), the mocks will always do it's thing.

1. yarn add express cors 
2. in the root of the project, create a directory called `real-api`
3. cd `real-api`
4. create a `index.js` file with the following content:

```
const express = require("express");
const cors = require("cors");

const port = 4000;
const corsOptions = {
  origin: "*",
  credentials: true, //access-control-allow-credentials:true
  optionSuccessStatus: 200,
};

const app = express();
app.use(cors(corsOptions)); // Allow all origins

// write a delay function
const delay = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

// Gives a random number from 1 to 10 inclusive
app.get("/api/number", async (req, res) => {
  await delay(500);
  res.json({ number: Math.floor(Math.random() * 10) + 1 });
});

app.listen(port, () => {
  console.log(`Real api listening on port ${port}`);
});

```

5. Open a new terminal and run the realapi in the background: `node real-api/index.js`
6. Adjust `redux/mySlice.ts`:

```
import { createAsyncThunk, createSlice } from "@reduxjs/toolkit";

export const fetchNumber = createAsyncThunk("number/random", async () => {
  const response = await fetch("http://localhost:4000/api/number/");
  return response.json();
});

interface CounterState {
  value: number;
  loading: boolean;
}

const initialState = { value: 0, loading: false };

const counterSlice = createSlice({
  name: "counter",
  initialState, // Delete this comment when tested. This used to be a cast but that is not desired here: initialState: initialState as CounterState,
  reducers: {
    increment: (state: CounterState) => {
      state.value += 1;
    },
  },
  extraReducers: (builder) => {
    builder.addCase(fetchNumber.fulfilled, (state, action) => {
      state.value = action.payload.number;
      state.loading = false;
    });
    builder.addCase(fetchNumber.pending, (state, action) => {
      state.loading = true;
    });
  },
});

export const { increment } = counterSlice.actions;

export default counterSlice.reducer;
```

7. Adjust `redux/App.tsx`:

```
import { fetchNumber, increment } from "./redux/mySlice";
import { useAppDispatch, useAppSelector } from "./redux/store";

function App() {
  const counter = useAppSelector((state) => state.mySlice.value);
  const loading = useAppSelector((state) => state.mySlice.loading);
  const dispatch = useAppDispatch();

  return (
    <>
      <button onClick={() => dispatch(increment())}>
        Click me to increment, {counter}
      </button>
      <button onClick={() => dispatch(fetchNumber())} disabled={loading}>
        Give random number from 1 to 10
      </button>
      <p>status: {loading ? "loading" : "idle"}</p>
    </>
  );
}

export default App;
```

The application uses the real api now, let's mock it!

8. yarn add msw --dev

9. Create a folder `utils` and within that create the file `test-utils.tsx` with the following content:
```
import React, { PropsWithChildren } from "react";
import { render } from "@testing-library/react";
import type { RenderOptions } from "@testing-library/react";
import { configureStore } from "@reduxjs/toolkit";
import type { PreloadedState } from "@reduxjs/toolkit";
import { Provider } from "react-redux";

import type { AppStore, RootState } from "../redux/store";
// As a basic setup, import your same slice reducers
import mySlice from "../redux/mySlice";

// This type interface extends the default options for render from RTL, as well
// as allows the user to specify other things such as initialState, store.
interface ExtendedRenderOptions extends Omit<RenderOptions, "queries"> {
  preloadedState?: PreloadedState<RootState>;
  store?: AppStore;
}

export function renderWithProviders(
  ui: React.ReactElement,
  {
    preloadedState = {},
    // Automatically create a store instance if no store was passed in
    store = configureStore({ reducer: { mySlice }, preloadedState }),
    ...renderOptions
  }: ExtendedRenderOptions = {}
) {
  function Wrapper({ children }: PropsWithChildren<{}>): JSX.Element {
    return <Provider store={store}>{children}</Provider>;
  }

  // Return an object with the store and all of RTL's query functions
  return { store, ...render(ui, { wrapper: Wrapper, ...renderOptions }) };
}
```

10. Create a file called `App.test.tsx` with the content:
```
import React from "react";
import { rest } from "msw";
import { setupServer } from "msw/node";
import { fireEvent, screen } from "@testing-library/react";
import { renderWithProviders } from "./utils/test-utils";
import App from "./App";

export const handlers = [
  rest.get("http://localhost:4000/api/number/", (req, res, ctx) => {
    return res(
      ctx.json({ number: Math.floor(Math.random() * 10) + 1 }),
      ctx.delay(500)
    );
  }),
];

const server = setupServer(...handlers);
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

const isLastNumberInRange = (testString: string) => {
  const matches = testString.match(/\d+$/); // Find the last sequence of digits at the end of the string
  if (matches) {
    const number = parseInt(matches[0]);
    if (number >= 1 && number <= 10) {
      return true;
    }
  }
  return false;
};

test("renders learn react link", async () => {
  renderWithProviders(<App />);
  let idleElement = await screen.getByText(/status: idle/i);
  expect(idleElement).toBeInTheDocument();

  fireEvent.click(
    screen.getByRole("button", { name: /Give random number from 1 to 10/i })
  );

  const loadingElement = await screen.findByText(/status: loading/i);
  expect(loadingElement).toBeInTheDocument();

  idleElement = await screen.findByText(/status: idle/i);
  expect(idleElement).toBeInTheDocument();

  const counterElement = screen.getByText(/Click me to increment/i);
  expect(counterElement).toBeInTheDocument();
  console.log("counterElement.textContent:", counterElement.textContent);
  expect(isLastNumberInRange(counterElement.textContent ?? "")).toBeTruthy();
});
```

11. Test it with `yarn test`

**Links** 
- https://mswjs.io/
- https://www.robinwieruch.de/react-testing-library/
- https://testing-library.com/docs/react-testing-library/intro/

# react-redux-RTL-mswjs-api-mock-typescript-easy-guide

This guide assumes create react app has been succesfully done, so follow this guide before getting into this one: https://github.com/marijndegen/redux-create-react-app-easy-guide

We will create a real api with nodejs (written in javascript) and adjust the react (typescript) so it consumes the real api. The tests will mock the real api using the mock service worker (MSW), so regardless of the http library (fetch, axios etc.), the mocks will always do it's thing.

1. yarn add express cors
2. in the root of the project, create a directory called `real-api`
3. cd `real-api`
4. create a `index.js` file with the followin content:

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
6. `yarn add msw --dev`
7. Adjust `redux/mySlice.ts`:

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
  initialState: initialState as CounterState,
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

8. Adjust `redux/App.tsx`:

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

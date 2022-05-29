# Frontend Architecture

Our frontend is using React 18. Each page is divided into functional components for modularization. The programming language to use is TypeScript as it's a better way of programming JavaScript programs. It provides typing, meaning that type correctness can be checked at compile time; and it points out compilation errors at development time.

## React 18

The Frontend project will have the following initial top-level directory structure:

- /src
  - /components
  - /hooks
  - /pages
    - index.tsx
  - /utils
  - App.tsx
- package.json

### /components

Components folder will have the reusable parts of a page. These can be, for example, buttons, input fields and containers.
Each component will live in a folder with it's name. It will follow the following pattern:

- components
  - Component
    - Component.tsx
    - Component.styles.ts
    - Component.test.ts
  - index.ts

The .tsx file will contain the actual component, whereas .styles.ts will contain the corresponding styles. The .test.ts component will remain for pertinent testing of the component.
The index.ts will take care of being the index of all components that are part of the directory.

### /hooks

The Hooks folder will have the reusable hooks of a component. As many components can share the same behavior, this behavior can be defined here. Mainly, it's proper meaning is to hide a hook functionality to simply have a variable and it's handler that depends on the state of this hook or another.
The /hooks directory structure can be similar to components where a reusable hook lives inside their subdirectories:

- /hooks
  - /useHookName
    - index.ts
    - useHookName.test.ts
  - index.js

### /pages

Pages directory will contain the UI for most of the application and the structure of each page in the form of React Functional Components. These components are naturally coupled with the business logic they represent. One way to separate business logic and the UI is to create a custom hook.

### /utils

If there are some global utility functions, they must be stored in the Utils directory. It contains app-wide constants and helper methods.

### Functional Components and React Hooks over Class Components

Functional Components are TypeScript functions. It's advantages over Class Components is that there is no render method. Also, with React Hooks, now, Functional Components can be stateful too, such as Class Components. The only main disadvantage is that there are no React lifecycle methods. This disadvantage mainly affects big projects, so it weights little over the other advantages.

Over all, Functional Components are better for writing code fast and without requiring to have big, complicated classes.

### NextJS

We chose NextJS because it loads only the TS and CSS that are needed for any given page, with SSR. This makes for much faster page loading times. It also provides a Head tag, to allow dynamic meta tags.

Another decisive advantage is that the router is not needed. It comes with it's own routers with no configuration. To access certain path, for example `/users/[user]`, one must only deposit the index.js file in the correct location. In this example, the page location would be the following:

- /pages
  - /users
    - [user].tsx

This can simplify the way someone reads the project structure.

## Docker

The frontend comes with a preconfigured file called `DOCKERFILE`. This file has all the instructions to automatically compile and create a docker image and nothing it should not be modified unless necessary.

### How to Develop

Let’s say we want to write a new page..

























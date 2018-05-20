### Pointers
1. Jest adds 'js-dom' by default and we can test javascript modules without any extra imports. Example: `window` object.


### Installation

1. npm install:
```javascript
npm install --save-dev jest
```
The reason we use `dev` is we are not interested in shipping the testing configurations to the prod and we shouldn't
so the configurations will only be added to `development`.

2. package.json
Adding test script to package.json,
```json
"scripts": {
    "test": "jest",
    "dev": "webpack-dev-server --mode=development",
    "build": "webpack --mode=production",
    "start": "serve -s dist --port 8080"
  },
```
3. webpack/babel problem

Error: `SyntaxError: Unexpected token import`.

The babel configurations set to `transpile` everything but `import` when using `webpack` because webpack supports
ESModules to achieve `tree shaking` which shrinks bundle the size by removing the `deadcode`. So the `import`s are
not transpiled and jest doesn't understand what to do with it. You can check the `package.json` file for `babel` selection
to understand the `presets`.

```javascript
// file: package.json ( this is not required in babel 7)
"babel": {
  "presets": "./.babelrc.js"
}
```
now the javascript file has the actual configurations
```javascript
// file: .babelrc.js
//added this to check if it's test and refer common.js else set to false to skip
const isTest = String(process.env.NODE_ENV) === 'test'
module.exports = {
  presets: [['env', {modules: isTest ? 'commonjs' : false}], 'react'],
  plugins: [
    'syntax-dynamic-import',
    'transform-class-properties',
    'transform-object-rest-spread',
  ],
}
```
4. jest config:

Error: `SyntaxError: Unexpected token .` ( pointing it to .css filename)

There are two ways we can refer `jest` configuration. We can add the configuration details for jest in package.json as mentioned below
```javascript
//file: package.json
"jest": {
  // skips js-dom installation but just node
  "testEnvironment": "node"
}
```
Or we can create `jest.config.js` and jest will pick it up by default
```javascript
// file:jest.config.js
module.exports ={
  moduleNameMapper: {
    // adds more details than empty module return in test
    '\\.module\\.css$': 'identity-obj-proxy',
    // node will not understand something ends with css so we need to define what to do
    '\\.css$': require.resolve('./test/style-mock')
  }
}
```
style-mock file has this line `module.exports = {}`. So if we execute test it will return just the empty `div` nothing else as our export is empty object. To
  over come this problem we have added `identity-obj-proxy` which also adds the `className` with `divs`.

5. Babel installs
```javascript
npm install --save-dev babel-plugin-dynamic-import-node
```
updated .babelrc.js looks like
```javascript
const isTest = String(process.env.NODE_ENV) === 'test'
module.exports = {
  presets: [['env', {modules: isTest ? 'commonjs' : false}], 'react'],
  plugins: [
    'syntax-dynamic-import',
    'transform-class-properties',
    'transform-object-rest-spread',
    isTest ? 'dynamic-import-node' : null
  ].filter(Boolean)
  /* letting dynamic import be null for testing by checking `isTest` and removing `null`
  values by using filter(Boolean).
  */
}
```

### Snippets

1. Form
Imagine we have a simple form which is intended to submit user login details then we would write the
testing as mentioned below,

1.1 Import
```JavaScript
import React from 'react';
import ReactDOM from 'react-dom';
import Login from '../login'; //component which we are intended to test
```

1.2 Bare minimum
```javascript
const onSubmit = jest.fn(); //mocking submit defined in the component
const container = document.createElement('div');
ReactDOM.render(<Login onSubmit={onSubmit}/>, container); // passing submit as props to the component while rendering
```

1.3 Fetching the form
```javascript
const form = container.querySelector('form'); // this will find the form dom element.
// console.log(form.elements.username)
const { username, password } = form.elements; // two elements defined in form i.e <Input name="username" />
```

1.4 Faking the data
```javascript
username.value="Tom";
password.value="123";
```

1.5 Submitting the form
```javascript
const submit = new window.Event('submit');  // make sure to use window.Event not window.event
form.dispatchEvent(submit);
```

1.6 Assertion or validation
```javascript
expect(onSubmit).toHaveBeenCalledTimes(1);
expect(onSubmit).toHaveBeenCalledWith({
   username: "Tom",
   password: "123"
 })
```

2. Mock API Call
If we have api call in our component then we should be testing the API call with mock to fake the process with defined/given data instead
of calling the actual api itself.

2.1 Import
```javascript
/**
 * the idea behind the below import is to reference the mock we will be creating later
 below by pointing out to the actual api. This will be figured out by jest to use the
 mock api call whenever we use utilMocks for our CRUD functions.
 */
import * as utilMocks from '../../utils/api';
```

2.2 Mock
```javascript
/**
MODULE DEPENDENCY: whenever we have module dependency we need to mock the module as mentioned below
instead of just faking it with `jest.fn()`. Actual api has create method for posts and hence we are
returning the create upon calling posts with resolved promise.
*/     
jest.mock('../../utils/api', () => {
  return {
    posts: {
      create: jest.fn((() => Promise.resolve()))
    }
  }
})
```
and this should be declared before executing the test case. For reference the posts in actual api
(i.e ../../utils/api) looks like below.
```javascript
const posts = {
  //... other methods which we are not interested now
  create: post => requests.post('/posts', post),
}
```

2.3 Usage
whenever we use the test to push data via component this mock will process the data and resolves the
request. Example of pushing the data is
```javascript
/* assertion to check if the api calls are submitted with the data */
expect(utilMocks.posts.create).toHaveBeenCalledTimes(1);
expect(utilMocks.posts.create).toHaveBeenCalledWith({
      title: title.value,
      date: expect.any(String)
    })
```

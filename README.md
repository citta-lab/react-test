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

### Using DOM nodes

Below example doesn't make use of any library we have out in the internet such as `enzyme` or `react-testing-library`
(abstraction).

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
const container = document.createElement('div'); // this will create document
ReactDOM.render(<Login onSubmit={onSubmit}/>, container); // passing submit as props to the component while rendering
```

1.3 Fetching the form
```javascript
/* this will find the form dom element */
const form = container.querySelector('form');
// console.log(form.elements.username)
const { username, password } = form.elements;
/**
* two elements defined in form i.e <Input name="username" />
* we retrieved form values by their name which is defined in the Input
*/
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

### Using Abstraction ( react-testing-library )

1. Form
Imagine we have a simple form which is intended to submit user login details then we would write the
testing as mentioned below,

1.1 Import
```JavaScript
import React from 'react';
import {render, Simulate} from 'react-testing-library'; // abstraction library
import Login from '../login'; //component which we are intended to test
```
note that we don't need to import ReactDOM from `react-dom` anymore.

1.2 Bare minimum
```javascript
const onSubmit = jest.fn(); //mocking submit defined in the component
/* In below step we will make use of elements from library render. */
const {container, getByLabelText, getByText} = render(<Login onSubmit={onSubmit}/>)
/* above snippet replaces the document creation and ReactDOM render */
```

1.3 Fetching the form
```javascript
const form = container.querySelector('form'); // this will find the form dom element.
// const { username, password } = form.elements;
const username = getByLabelText('Username')
const password = getByLabelText('Password')
// these are case insensitive
```

1.4 Faking the data
```javascript
username.value="Tom";
password.value="123";
```

1.5 Submitting the form
```javascript
const submitButton = getByText('Submit') // added to find the button
Simulate.submit(form);
/* instead of creating window.Event and dispatchEvent we make use of react Simulate.
   here `form` is from step 1.3
*/
```

1.6 Assertion or validation
```javascript
expect(onSubmit).toHaveBeenCalledTimes(1);
expect(onSubmit).toHaveBeenCalledWith({
   username: "Tom",
   password: "123"
 })
  expect(submitButton.type).toBe('submit')
```
At this point we should have our testing passing however if we want to refactor the code little more to simplify then
we can use `renderIntoDocument` instead of `render` from `react-testing-library`. This way we don't need to use
reacts `Simulate` method to submit the form. So the changes would look like below,

```javascript
import {renderIntoDocument, cleanup} from 'react-testing-library'; // abstraction library
afterEach(cleanup);  // use this to clean up the document element.
//... rest of the code
const {container, getByLabelText, getByText} = renderIntoDocument(<Login onSubmit={onSubmit}/>)
// ... remaining code
// remove Simulate.submit(form);
submitButton.click()
```

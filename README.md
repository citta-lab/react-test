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
const isTest = String(process.env.NODE_ENV) === 'test' //added this to check if it's test and refer common.js else set to false to skip
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
  "testEnvironment": "node" // skips js-dom installation but just node
}
```
Or we can create `jest.config.js` and jest will pick it up by default
```javascript
// file:jest.config.js
module.exports ={
  moduleNameMapper: {
    '\\.module\\.css$': 'identity-obj-proxy', // adds more details than empty module return in test
    '\\.css$': require.resolve('./test/style-mock') // node will not understand something ends with css so we need to define what to do
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
}
```

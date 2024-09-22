# stream-json-concept-examples

stream-json github: https://github.com/uhop/stream-json

## Example 1

```js
import fs from 'fs';
import Pick from 'stream-json/filters/Pick.js';
import Parser from 'stream-json/Parser.js';
import StreamArray from 'stream-json/streamers/StreamArray.js';
import StreamObject from 'stream-json/streamers/StreamObject.js';
import Chain from 'stream-chain';

const {pick} = Pick;
const {parser} = Parser;
const {streamArray} = StreamArray;
const {streamObject} = StreamObject;
const {chain} = Chain;

// [^.]+   : anything thats not a dot
// \w+     : all word characters
// \d+     : all digits

// const filePath = 'sample.json';
// const filePath = 'sample-object.json';

// This class reads and operates on the stream in a single pass,
//  meaning that only one stream function (find, forEach etc) can be used per stream.
class JSONStream {
  #mode;
  #regex;
  #regexString;

  constructor(filePath) {
    this.filePath = filePath;
  }

  #select(matchString) {
    this.#regexString = '^';
    const tokens = matchString.split('.');
    
    for (let i = 0; i < tokens.length; i++) {
      const token = tokens[i];

      if (i > 0) {
        this.#regexString += '\\.';
      }

      if (token === '${index}') {
        this.#regexString += '\\d+';
      }
      else if (token === '${key}') {
        this.#regexString += '[^.]+';
      }
      else {
        this.#regexString += token;
      }
    }

    this.#regexString += '$';
    this.#regex = new RegExp(this.#regexString);
    return this;
  }

  // For utility usage only
  #unselect() {
    this.#regex;
    this.#regexString;
    return this;
  }

  // Turn into entry by entry stream array iterator.
  // Omitting this would return the arrays
  array(propertyPath = '') {
    this.#mode = streamArray;
    // propertyPath += '.${index}';
    this.#select(propertyPath);
    return this;
  }

  object(propertyPath = '') {
    this.#mode = streamObject;
    // propertyPath += '.${key}';
    this.#select(propertyPath);
    return this;
  }

  // Callback is called with 2 arguments (value, index/key)
  forEach(callback) {
    if (this.#mode != streamArray && this.#mode != streamObject) {
      throw new Error(`Stream-mode must be set to array or object.`);
    }

    this.pipeline = chain([
      fs.createReadStream(filePath),
      parser(),
      pick({filter: this.#regex}),
      this.#mode(),
      // streamValues(),
      // callback
    ]);

    this.pipeline.on('data', (data) => {
      callback(data.value, data.key);
    });

    this.pipeline.on('end', () => {
      this.#mode = undefined;
      this.#unselect();
      console.log('Finished processing the entire JSON structure.');
    });
    
    this.pipeline.on('error', (err) => {
      console.error('Error during processing:', err);
    });
  }
  
  find(callback) {
    // Find only makes sense for arrays
    if (this.#mode !== 'array') {
      throw new Error(`Stream-mode must be set to array.`);
    }

    return new Promise((resolve) => {
      this.forEach((data) => {
        if (callback(data)) {
          resolve(data);
        }
      });
    });
  }
}
```

## Example 1: Usage

### find method

```js
const json = new JSONStream('sample.json');

const found = await json.array('results.data.${index}.items').find((data) => {
  return data.item_id === 'A2';
});

console.log(found);
```

### forEach method

```js
const json = new JSONStream('sample.json');

json.array('results.data.${index}.items').forEach((data, index) => {
  console.log("Index: " + index + ", Value: " + JSON.stringify(data));
});
```

## Example 1: Notes

The stream-mode must be set first by calling the array method or the object method before the methods above are called. A stream-mode method and a method from the above usage examples can be chained.

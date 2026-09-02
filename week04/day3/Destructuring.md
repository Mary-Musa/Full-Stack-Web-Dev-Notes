# JavaScript Destructuring 

## Learning Objectives

By the end of this lesson, you will be able to:
- Extract properties from objects into distinct variables.
- Rename destructured variables to avoid naming collisions.
- Assign default values during destructuring to handle `undefined` safely.
- Use the rest syntax (`...`) to collect remaining object properties and array elements.
- Unpack values from arrays by position.
- Destructure objects directly within function parameters.

---

## 1. Object Destructuring

Object destructuring allows you to unpack properties from an object and assign them directly to variables using their property names.

### Basic Extraction
```javascript
const user = {
  name: "Alice",
  age: 20
};

// Extracts 'name' and 'age' directly from user
const { name, age } = user;

console.log(name); // "Alice"
console.log(age);  // 20
```
### 1.1 Renaming Variables
If you want the extracted variable to have a different name than the object key:
```js
const user = {
  name: "Alice",
  age: 20
};

// 'name' from user is assigned to a new variable called 'userName'
const { name: userName } = user;

console.log(userName); // "Alice"
```
### 1.2 Default Values
If a property might not exist (evaluates to `undefined`), you can supply a fallback value:
```js
const user = {
  name: "Alice"
};

// Falls back to "Kenya" if country is undefined
const { country = "Kenya" } = user;

console.log(country); // "Kenya"
```
You can also combine renaming with default values:
```js
const product = { name: "Pen" };
const { name: productName, inStock = false } = product;

console.log(productName); // "Pen"
console.log(inStock);     // false
```
### 1.3 Rest Operator with Objects
Collecting the remaining properties into a new object using `...`
```js
const user = {
  name: "Alice",
  age: 20,
  city: "Nairobi",
  role: "Student"
};

const { name, ...otherDetails } = user;

console.log(name);         // "Alice"
console.log(otherDetails); // { age: 20, city: "Nairobi", role: "Student" }
```
## 2. Array Destructuring
Array destructuring unpacks values based on index position rather than property names.

### 2.1 Positional Unpacking
```js
const numbers = [10, 20, 30];

const [first, second, third] = numbers;

console.log(first);  // 10
console.log(second); // 20
console.log(third);  // 30
```

### 2.2 Skipping Elements & Default Values
```js
const rgb = [255, 128];

// Skipping index 1 and applying a default to index 2
const [red, , blue = 0] = rgb;

console.log(red);  // 255
console.log(blue); // 0
```

### 2.3 Rest Operator with Arrays
We collect all trailing elements into a new array
```js
const numbers = [10, 20, 30, 40, 50];

const [first, ...remaining] = numbers;

console.log(first);     // 10
console.log(remaining); // [20, 30, 40, 50]
```

# 3. Parameter Destructuring
Instead of passing an object and accessing properties via dot notation inside the function, destructure directly in the parameter signature:
```js
function summarise({ name: productName, priceCents, inStock = false }) {
  const status = inStock ? "available" : "out of stock";
  return `${productName}: ${status} at KSh ${priceCents / 100}`;
}

const item = { name: "Notebook", priceCents: 15000, inStock: true };
console.log(summarise(item));
// "Notebook: available at KSh 150"

console.log(summarise({ name: "Pen", priceCents: 5000 }));
// "Pen: out of stock at KSh 50" (uses default inStock = false)
```

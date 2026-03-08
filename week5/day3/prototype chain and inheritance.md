# Prototype Chain & Inheritance

## Objectives
- Explain how JavaScript’s prototype chain drives property lookup and inheritance.
- Inspect and traverse prototypes via `__proto__` and `Object.getPrototypeOf()`.
- Augment existing prototypes to share methods across instances.
- Use `Object.create()` to establish inheritance without constructors.
- Be cautious of modifying built-in prototypes (monkey-patching risks).

---

## 1. Prototype Chain Fundamentals
Every JavaScript object has an internal link, its **prototype**, to another object. When you access a property:
1. JS looks on the object itself.
2. If not found, it follows the internal `[[Prototype]]` (exposed as `__proto__` or via `Object.getPrototypeOf`).
3. It continues up the chain until it either finds the property or reaches `null` (the end).

```javascript
const a = { x: 1 };
const b = Object.create(a); // b.[[Prototype]] → a
b.y = 2;

console.log(b.x); // 1  ← found on a
console.log(b.y); // 2  ← found on b
console.log(b.z); // undefined ← reached end of chain
```

> [!TIP]
> This lets us share methods/data on one prototype object and have all instances inherit them without duplicating per-instance.

---

## 2. Inspecting & Traversing the Chain
You can inspect an object's prototype using:
- `obj.__proto__` (non-standard but ubiquitous)
- `Object.getPrototypeOf(obj)` (standard)

```javascript
function Gadget(name) {
  this.name = name;
}

Gadget.prototype.getName = function() {
  return this.name;
};

const phone = new Gadget('Phone');
console.log(phone.getName());            // "Phone"
console.log(Object.getPrototypeOf(phone) === Gadget.prototype); // true
console.log(Object.getPrototypeOf(Gadget.prototype) === Object.prototype); // true
console.log(Object.getPrototypeOf(Object.prototype)); // null (Chain ends here)
```

---

## 3. Adding Methods at Runtime
You can augment a prototype after instances already exist, those instances immediately “see” the new method:

```javascript
function Person(name) { this.name = name; }
const p1 = new Person('Alice');

Person.prototype.introduce = function() {
  console.log(`Hi, I’m ${this.name}`);
};

p1.introduce(); // works, logs "Hi, I’m Alice"
```

> [!WARNING]
> Prefer defining prototypes once (e.g., during class or constructor definition) rather than dynamically monkey-patching at runtime.

---

## 4. Built-in Prototype Extension (With Caution)
JavaScript lets you extend prototypes of built-in types like `Array` or `String`, but this is generally discouraged (**Monkeypatching**).

```javascript
// Add `last()` to all arrays
Array.prototype.last = function() {
  return this[this.length - 1];
};

console.log([1,2,3].last()); // 3
```

**Why is this risky?**
- **Naming Conflicts**: A future JS update might add a native `last()` method with different behavior.
- **Maintainability**: Other developers or libraries might be surprised by modified global behavior.

---

## 5. Inheritance with `Object.create`
`Object.create(proto)` creates a new object whose prototype is set to `proto`.

```javascript
const animal = {
  eats: true,
  walk() {
    console.log('Walking…');
  }
};

const rabbit = Object.create(animal);
rabbit.jumps = true;

console.log(rabbit.eats); // true (inherited)
rabbit.walk();            // Walking…
```

You can even pass a second argument to define properties:

```javascript
const dog = Object.create(animal, {
  bark: {
    value() { console.log('Woof'); },
    enumerable: true
  }
});
dog.bark(); // Woof
```

---

## 6. Property Lookup & Shadowing
If an object defines its own property with the same name as one on its prototype, the **own property shadows** the prototype’s:

```javascript
const parent = { value: 1 };
const child  = Object.create(parent);
child.value = 2;

console.log(child.value); // 2 (shadows prototype)
delete child.value;
console.log(child.value); // 1 (fallback to prototype)
```

---

## 🚀 Deep Dive: Advanced Concepts

### `Object.setPrototypeOf` vs. `Object.create`
While `Object.setPrototypeOf(obj, newProto)` exists, it is **extremely slow** because it forces the JS engine to discard optimizations made for that object's "shape."
- **Best Practice**: Use `Object.create()` to set the prototype at creation time.

### `hasOwnProperty` vs. `in` Operator
- **`in`**: Returns `true` if the property exists on the object **or anywhere in its prototype chain**.
- **`hasOwnProperty()`**: Returns `true` **only** if the property is defined on the object itself.

```javascript
const user = Object.create({ isAdmin: true });
user.name = "Alice";

console.log("isAdmin" in user);      // true
console.log(user.hasOwnProperty("isAdmin")); // false
```

### `instanceof` Internals
The `instanceof` operator doesn't check the constructor function itself; it checks if the **constructor's `prototype` object** exists anywhere in the object's prototype chain.

---

## 7. In-Class Activity

### Enhance the Person prototype
1. Define function `Person(name)`.
2. Add a method `describe()` on `Person.prototype` so that `alice.describe()` logs `"Person: Alice"`.

```javascript
function Person(name) {
  this.name = name;
}
const alice = new Person('Alice');
const bob   = new Person('Bob');

// Task: add a method `describe()` on Person.prototype
// so that alice.describe() logs "Person: Alice"
```

### Prototype Checks
1. Use `Person.prototype.isPrototypeOf(alice)` (expected: `true`).
2. Use `alice.hasOwnProperty('name')` vs `alice.hasOwnProperty('describe')`.

### Build a Shape Chain
1. Create a `Shape` object with `type: 'shape'`.
2. Use `Object.create(Shape)` to create a `Circle` with a `radius` and an `area()` method.

```javascript
const Shape = { type: 'shape' };
const Circle = Object.create(Shape, {
  radius: { value: 1, writable: true, enumerable: true },
  area: {
    value() { return Math.PI * this.radius ** 2; },
    enumerable: true
  }
});

const c = Object.create(Circle);
c.radius = 5;
console.log(c.type);      // 'shape'
console.log(c.area());    // 78.5398…
```

---

## 8. Assignment
### Add `last()` to Arrays (Non-Enumerable)
Implement `Array.prototype.last` but ensure it doesn't show up in `for...in` loops:
```javascript
Object.defineProperty(Array.prototype, 'last', {
  value() { return this[this.length - 1]; },
  enumerable: false
});
```

Test with:

```javascript
console.log([].last());          // undefined
console.log([1].last());         // 1
console.log([1,2,3,4].last());   // 4
```

Demonstrate property shadowing:

```javascript
const arr = [1,2,3];
arr.last = () => 'hijacked';
console.log(arr.last()); // 'hijacked' (own)
delete arr.last;
console.log(arr.last()); // 3       (prototype)
```

Stretch:

- Using `Object.create`, build a Stack object whose prototype has `push`, `pop`, and `peek` methods, and whose instances maintain their own internal array.

---

*With a deep understanding of prototype chains and inheritance patterns, you’re now ready for ES6 modules and modern code organization.*
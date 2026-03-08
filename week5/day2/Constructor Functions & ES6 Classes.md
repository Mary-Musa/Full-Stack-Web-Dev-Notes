# Constructor Functions & ES6 Classes

## Objectives
- Define object **blueprints** using constructor functions and ES6 class syntax.
- Understand how the `new` operator creates instances and sets up the prototype chain.
- Attach methods via **prototype** (constructor functions) vs. **class bodies** (ES6).
- Implement inheritance both manually and with `extends`/`super`.
- Compare the two approaches and recognize underlying prototype-based behavior.

---

## 1. Constructor Functions
Before ES6 class syntax, JavaScript relied on functions to act as constructors, blueprints for creating multiple similar objects.

### 1.1 Defining a Constructor
```javascript
function Person(name, age) {
  // When invoked with `new`, `this` is a fresh object
  this.name = name;
  this.age  = age;
}

// Add shared methods on the prototype
Person.prototype.greet = function() {
  console.log(`Hi, I’m ${this.name} and I’m ${this.age} years old.`);
};

// Create instances
const alice = new Person('Alice', 30);
const bob   = new Person('Bob',   25);

alice.greet(); // "Hi, I’m Alice and I’m 30 years old."
bob.greet();   // "Hi, I’m Bob and I’m 25 years old."
```

### 1.2 How `new` Works (Behind the Scenes)
When you call `new Person('Alice', 30)`, JavaScript does:
1. Creates a brand-new empty object: `{}`.
2. Links its internal prototype (`[[Prototype]]`) to `Person.prototype`.
3. Binds `this` inside `Person()` to that new object.
4. Executes the constructor body (`this.name = name`, etc.).
5. Returns the newly created object (unless the constructor returns its own object).

This mechanism sets up the **prototype chain**, so any lookup for a missing property on the instance goes up to `Person.prototype`.

### 1.3 Pitfalls of Constructor Functions
- **Forgetting `new`**: Calling `Person('Alice', 30)` without `new` binds `this` to the global object (or `undefined` in strict mode).
- **Manual prototype tweaks**: To extend or inherit, you must explicitly set up prototypes and constructors, which can be verbose.

---

## 2. ES6 Classes
ES6 introduced `class` syntax as **syntactic sugar** over prototype-based inheritance. Under the hood, it works the same, but the code is clearer.

### 2.1 Defining a Class
```javascript
class Person {
  // The constructor method runs when you call `new Person()`
  constructor(name, age) {
    this.name = name;
    this.age  = age;
  }

  // Methods defined here go on Person.prototype
  greet() {
    console.log(`Hi, I’m ${this.name}, age ${this.age}.`);
  }

  // Static methods live on the constructor itself
  static species() {
    return 'Homo sapiens';
  }
}

const charlie = new Person('Charlie', 28);
charlie.greet();             // "Hi, I’m Charlie, age 28."
console.log(Person.species()); // "Homo sapiens"
```

### 2.1 Class Characteristics
- **Not hoisted**: Unlike function declarations, classes must be defined before use.
- **Strict mode**: Class bodies always run in strict mode.
- **Flat syntax**: No `function` keyword inside; methods are concise.

---

## 3. Inheritance

### 3.1 Constructor Functions Pattern
```javascript
function Student(name, age, course) {
  Person.call(this, name, age); // Call parent constructor
  this.course = course;
}

// Link prototypes
Student.prototype = Object.create(Person.prototype);
// Restore constructor reference
Student.prototype.constructor = Student;

Student.prototype.study = function() {
  console.log(`${this.name} is studying ${this.course}.`);
};
```

### 3.2 ES6 `extends` / `super`
```javascript
class Student extends Person {
  constructor(name, age, course) {
    super(name, age); // Must call super() before using `this`
    this.course = course;
  }

  study() {
    console.log(`${this.name} is studying ${this.course}.`);
  }
}
```

---

## Deep Dive: Advanced Concepts

### Prototype vs. Instance Methods
Why do we attach methods to the `prototype` instead of putting them inside the constructor?
- **Instance Method**: If you define `this.greet = function() { ... }` inside the constructor, every instance gets its own copy of the function. For 1,000 objects, you have 1,000 function instances in memory.
- **Prototype Method**: Defining `Person.prototype.greet` means all instances share a **single** function reference. This is much more memory-efficient.

### Private Fields (`#`)
Modern JavaScript allows private fields that cannot be accessed or modified from outside the class.
```javascript
class User {
  #password; // Private field

  constructor(username, password) {
    this.username = username;
    this.#password = password;
  }

  checkPassword(input) {
    return input === this.#password;
  }
}
```

### The Prototype Chain
Every object in JS has a prototype. When you call a method:
1. JS looks at the **instance** itself.
2. If not found, it looks at the **prototype** (e.g., `Person.prototype`).
3. If still not found, it keeps going up the chain (e.g., `Object.prototype`) until it reaches `null`.

---

## 4. In-Class Activity

### Constructor Function Exercise
1. Define function `Car(make, model, year)`.
2. Add `getInfo()` to `Car.prototype` returning `"2018 Toyota Camry"`.
3. Instantiate two cars and log their `getInfo()`.

### Convert to ES6 Class
1. Rewrite `Car` as a class.
2. Add a static method `Car.compareAge(carA, carB)` that returns the newer model.

### Inheritance
1. Create `ElectricCar extends Car` with `batteryRange`.
2. Override `getInfo()` to include battery range (e.g., `"2020 Tesla Model 3 – 322 mi range"`).

---

## 5. Assignment
### User & Admin Hierarchy
1. Create a class `User` (constructor with `username`, `email`; `describe()` method; static `validateEmail(email)`).
2. Create `Admin extends User` (adds `permissions`; `showPermissions()` method; overrides `describe()`).
3. **Stretch**: Use private fields (`#email`) and a getter.

---

*By completing this lesson, you’ll be fluent in both classical constructor-pattern and modern class-based inheritance, essential for modeling complex domains in JavaScript.*
 
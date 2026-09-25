# Solidity Syntax Primer

An introductory syntax reference for Solidity, written as a bridge from JavaScript (and, where it still helps, Python) — for anyone starting an EVM bootcamp with a JS/web background and no prior Solidity exposure.

Solidity looks like JS at a glance (curly braces, semicolons, C-style syntax) but it's a fundamentally different kind of language underneath: it's **statically typed**, **compiled**, and everything it does costs real money (gas) and runs on a public, immutable ledger. Keep that in mind — syntax similarities to JS are surface-level; the execution model is not like JS or Python at all.

---

## 0. First, About That `mapping` Line

Since this is what prompted this doc — let's kill the confusion immediately:

```solidity
mapping(address => uint) balances;
```

This is **not** an arrow function, and it's **not** related to JS's `.map()` array method either, despite the shared word "map." `mapping` is Solidity's built-in **key-value store type** — closest to a JS `Map`/`Object` or a Python `dict`. The `=>` here means "maps to," describing the *type relationship* between keys and values — it is declaring a type, not executing any logic.

| | Solidity | JS equivalent (closest concept) | Python equivalent (closest concept) |
|---|---|---|---|
| Declare | `mapping(address => uint) public balances;` | `const balances = {};` / `new Map()` | `balances = {}` |
| Read | `balances[someAddress]` | `balances[someAddress]` | `balances[some_address]` |
| Write | `balances[someAddress] = 100;` | `balances[someAddress] = 100;` | `balances[some_address] = 100` |
| Default value for a missing key | Returns the type's zero-value (`0` for `uint`, `false` for `bool`, etc.) — **never throws** | `undefined` | Raises `KeyError` |
| Can you iterate over all keys? | **No** — no built-in way to list keys or check "does this key exist" | Yes (`Object.keys()`, `for...in`) | Yes (`.keys()`, `for k in dict`) |

That last row is the real gotcha to internalize early: a Solidity `mapping` is deliberately *not* enumerable. This is a gas/storage design decision, not a missing feature — Solidity avoids operations whose cost scales with unknown data size. If you need to list all keys, you maintain a separate array of keys alongside the mapping yourself — a very common pattern you'll see everywhere in real contracts.

---

## 1. File Structure Basics

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract SimpleStorage {
    uint public storedValue;

    function set(uint _value) public {
        storedValue = _value;
    }

    function get() public view returns (uint) {
        return storedValue;
    }
}
```

| Line | Purpose |
|---|---|
| `// SPDX-License-Identifier: MIT` | A machine-readable license comment — expected convention, not optional in practice |
| `pragma solidity ^0.8.20;` | Declares which compiler version(s) this file is written for — Solidity has no runtime, so this matters a lot more than a JS/Python version comment ever would |
| `contract SimpleStorage { }` | Roughly analogous to a JS/Python `class` — a contract is a blueprint that gets deployed as a standalone, permanent object on the blockchain |

> There's no `import`-everything-by-default or REPL-style scripting here — every file is compiled ahead of time, and a `contract` isn't instantiated locally like a JS/Python object; it's **deployed**, gets its own permanent address, and (usually) can't be edited afterward.

---

## 2. Variables & Types

Unlike JS (`let`/`const`, dynamically typed) or Python (no declaration, dynamically typed), Solidity requires an explicit type for every variable — closer in spirit to TypeScript than to plain JS.

| Type | Meaning | Rough JS/Python equivalent |
|---|---|---|
| `uint` / `uint256` | Unsigned integer (no negatives) — `uint` is shorthand for `uint256` | `number` (JS) / `int` (Python), but unsigned-only and fixed-width |
| `int` / `int256` | Signed integer | `number` / `int` |
| `bool` | `true` / `false` | `boolean` / `bool` (note: lowercase `true`/`false` like JS, not `True`/`False`) |
| `address` | A 20-byte Ethereum address — a distinct, built-in type, not just a string | No real equivalent — closest is "a specially validated string" |
| `string` | UTF-8 text | `string` / `str` |
| `bytes` / `bytes32` | Raw byte data, fixed or dynamic length | No direct equivalent — closest is a `Buffer` (Node) or Python `bytes` |

```solidity
uint public age = 30;
bool public isActive = true;
address public owner = 0x1234567890AbcdEF1234567890aBcdef12345678;
string public name = "Andres";
```

> **No type inference, no `var`/`let`/`const`-style flexibility:** once you declare `uint age`, `age` can only ever hold a `uint`. There's no equivalent to JS reassigning a variable to a different type, or Python's fully dynamic typing.

---

## 3. State Variables vs Local Variables (New Concept — No Direct JS/Python Equivalent)

This distinction doesn't really exist in JS or Python and is worth calling out explicitly, because it maps directly to cost and permanence.

| | State variable | Local variable |
|---|---|---|
| Declared | Outside any function, at the contract level | Inside a function body |
| Storage | Permanently written to the blockchain (`storage`) | Temporary, exists only during function execution (`memory`) |
| Cost | Writing to it costs real gas — sometimes a lot | Cheap — memory is temporary and much cheaper than storage |
| Closest analogy | A class's instance attribute that's saved to a permanent database on every write | An ordinary local variable inside a JS/Python function |

```solidity
contract Example {
    uint public total;   // state variable — lives on-chain permanently

    function addToTotal(uint amount) public {
        uint doubled = amount * 2;  // local variable — gone after this function returns
        total += doubled;            // this line costs gas; the line above barely does
    }
}
```

---

## 4. Functions

```solidity
function transfer(address to, uint amount) public returns (bool) {
    balances[msg.sender] -= amount;
    balances[to] += amount;
    return true;
}
```

| Piece | JS/Python comparison |
|---|---|
| `function transfer(address to, uint amount)` | Like `function transfer(to, amount)` (JS) / `def transfer(to, amount):` (Python), but every parameter needs an explicit type |
| `public` | **Visibility modifier — new concept.** No JS/Python equivalent at the language level (closest: Python's `_`/`__` naming convention, but that's a much weaker guarantee — see §5) |
| `returns (bool)` | Like a JS/Python `return`, but the return *type* must be declared up front in the function signature |

### Visibility Modifiers (no real JS/Python equivalent)

| Modifier | Who can call it |
|---|---|
| `public` | Anyone — other contracts, external accounts, and internally |
| `private` | Only from within this exact contract |
| `internal` | This contract and any contract that inherits from it |
| `external` | Only from *outside* the contract — cannot be called internally without extra syntax |

### State Mutability Modifiers (also a new concept)

| Modifier | Meaning |
|---|---|
| `view` | Reads state but doesn't modify it — free to call, costs no gas when called externally |
| `pure` | Doesn't read *or* write state — a pure computation, like a pure function in the functional-programming sense |
| `payable` | Can receive ETH when called |
| *(none)* | Can modify state — costs gas |

```solidity
function getBalance() public view returns (uint) {
    return balances[msg.sender]; // reads state, doesn't change it → view
}

function add(uint a, uint b) public pure returns (uint) {
    return a + b; // touches no state at all → pure
}
```

> **Web3-relevant mental shift:** in JS/Python, whether a function "costs" anything is invisible to the language itself. In Solidity, `view`/`pure` vs a regular state-changing function is a *visible, declared* distinction — because one costs gas and the other doesn't.

---

## 5. Special Global Variables (No Equivalent At All)

These exist because a Solidity function call carries blockchain-specific context that a JS/Python function call simply never has.

| Variable | Meaning |
|---|---|
| `msg.sender` | The address that called this function |
| `msg.value` | How much ETH (in wei) was sent with this call |
| `block.timestamp` | The current block's timestamp — closest thing to `Date.now()`, but it's the *block's* time, set by a miner/validator, not your local clock |
| `address(this)` | This contract's own address |

---

## 6. Structs and Arrays

```solidity
struct User {
    string name;
    uint balance;
}

User[] public users;                          // dynamic array
mapping(address => User) public userByAddress; // mapping to a struct

users.push(User("Andres", 100));
```

| | Solidity | JS equivalent | Python equivalent |
|---|---|---|---|
| Group of named fields | `struct` | plain object `{ name, balance }` | class or `dict` |
| Ordered collection | `User[]` / `uint[3]` (fixed-size) | `Array` | `list` |
| Add an item | `.push(...)` | `.push(...)` | `.append(...)` |

---

## 7. Error Handling: `require` / `revert` / `assert`

Solidity has no `try`/`catch`/`throw` in the JS/Python sense for normal control flow (there is a `try`/`catch` construct, but it's narrowly used for calling *other* contracts, not everyday validation).

```solidity
function withdraw(uint amount) public {
    require(amount <= balances[msg.sender], "Insufficient balance"); // most common — validate input/conditions
    balances[msg.sender] -= amount;
}
```

| Keyword | Use case | Effect on failure |
|---|---|---|
| `require(condition, "message")` | Validating inputs, permissions, conditions — the everyday workhorse | Reverts the transaction, refunds remaining gas, returns the message |
| `revert("message")` | Same effect as `require`, used when the logic doesn't fit a simple condition check | Same as above |
| `assert(condition)` | Checking for conditions that should be **mathematically impossible** — a failure here signals a bug in the contract itself | Reverts, consumes all remaining gas (historically; behavior has evolved by compiler version) |

> **Key mental shift:** a failed `require` doesn't just return an error value or throw a catchable exception the way JS/Python would — it **reverts the entire transaction**, as if it never happened. Any state changes made earlier in that same function call are undone too.

---

## 8. Inheritance

```solidity
contract Ownable {
    address public owner;
    constructor() {
        owner = msg.sender;
    }
}

contract MyToken is Ownable {
    // inherits "owner" and the constructor logic
}
```

| | Solidity | JS |
|---|---|---|
| Inherit | `contract Child is Parent { }` | `class Child extends Parent { }` |
| Constructor | `constructor() { }` (same keyword as JS) | `constructor() { }` |
| Call parent constructor | `constructor() Ownable() { }` | `super();` |

---

## 9. Quick Cheat Sheet

| Concept | Solidity | JS equivalent |
|---|---|---|
| Key-value store | `mapping(address => uint)` | `{}` / `Map` |
| Class-like blueprint | `contract` | `class` |
| Declare with type | `uint x = 5;` | `let x = 5;` *(no type)* |
| Function | `function foo() public returns (uint) {}` | `function foo() {}` |
| Group of fields | `struct` | plain object |
| List | `uint[] public list;` | `Array` |
| Validate a condition | `require(cond, "msg");` | `if (!cond) throw new Error("msg");` |
| Inheritance | `contract B is A {}` | `class B extends A {}` |
| Caller's address | `msg.sender` | *(no equivalent)* |
| ETH sent with call | `msg.value` | *(no equivalent)* |
| Permanent on-chain data | state variable | *(no equivalent — closest is a database row)* |

---

### Key Takeaway

> Solidity borrows JS's *look* (braces, semicolons, `function`, `contract...is...` echoing `class...extends`) but not its *behavior*. The concepts that don't map to anything in JS or Python — visibility modifiers, `view`/`pure`, `msg.sender`, the storage-vs-memory cost distinction, and `require`'s whole-transaction revert — are exactly the parts worth studying deliberately rather than pattern-matching from prior languages, since guessing by syntax resemblance alone is exactly what turned `mapping(address => uint)` into an apparent arrow function at first glance.

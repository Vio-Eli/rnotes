## Idea:
- Idea is to change Add and Mul to `Vec<Arc<Function>>` to simplify needless recursion
	- e.g. OLD:  `x + y + z + a -> Add(Add(x, y), Add(z, a))`
	- NEW: `x + y + z + a -> Add([x, y, z, a])`
- This is good for 2 reasons
	1. Functions like `(x + y) + z` and `x + (y + z)` are now easily equitable
		- The current (old) structure doesn't represent commutativity well
	2. It makes a basic simplify *much* easier

## Simplify:
- Using a hashmap as a lookup tree, we can efficiently flatten and simplify the tree
- Returns a *simplified* Function
	- Uses Simplify type in the enum
		- Simplify Type is marked for removal, so we will need to create some wrapper just for simplify
## Freeze:
- Introduced 5/15/24
- Deterministic UUID for each Function that understands Associativity and Commutativity for Add and Mul
	- Also replaces current method for PartialEQ
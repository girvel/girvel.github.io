# Concatenation is not addition -- my love letter to Lua

Traditionaly you can join two strings with the operator `+` in almost every modern programming language, and there is no reason for that. Concatenation has very little in common with addition and there is no need to merge together two separate operations.

1. Addition is commutative: `1 + 2 == 2 + 1`. Concatenation is not: `"a" .. "b" != "b" .. "a"`.
2. Addition has the inverse operation `-`, concatenation does not.
3. Considering only the primitive types, you can add only the numbers (and maybe booleans), but you can concatenate anything with anything. Every year hundreds of JavaScript developers suffer rage attacks, because they forgot to cast one variable in that one place and now `norm + i + 1` is 1121 instead of 14 and the UI is fucked.

So, the conclusion here is: I love Lua, and Javascript is terrible. Also, why array.sort in Javascript sorts integers alphabetticaly? What the fuck?

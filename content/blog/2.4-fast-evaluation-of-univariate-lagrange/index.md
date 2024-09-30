---  
title: Fast Evaluation of Univariate Lagrange Interpolating Polynomials  
date: "2024-07-03T06:26:45.934Z"  
description: "Optimization that reduces evaluation of a univariate low degree extension at a field element from $O(n^2)$ to $O(n)$ time"
---  
# Naive Evaluation  in $O(n^2)$
- Given a vector $m \in \mathbb{F}_p^n$ with univariate low degree extension $q_m$, evaluating $q_m(r)$ for $r \in \mathbb{F}_p$ naively would require $O(n^2)$ time.
- The interpolating polynomial $q_m$ has n terms  
  $$  
  q_m(r) = \sum^{n-1}_{i=0}m_{i+1}\cdot L_i(r)  
  $$
  but each Lagrange basis polynomial $L_i(r)$ has another $n-1$ terms  
  $$  
  L_i(r) = \prod^{n-1}_{j=0;j\neq i} \frac{x-x_j}{x_i-x_j}  
  $$
- To compute $q_m(r)$ according to these definitions, we would have $O(n\cdot(n-1)) \approx O(n^2)$ operations to perform.
# Linear Time Shortcut
- We can eliminate most of the work associated with computing each Lagrange basis polynomial by computing the basis polynomials recursively.
- We rely on the fact that since the nodes over which our Lagrange basis is defined are the same as their indices ($x_i = i$), the $i$th Lagrange basis polynomial can be expressed as a function of the $i-1$th basis polynomial which can be computed in constant time.
  - This optimization only applies for an interpolating set where the entries are the same as the indices: $\{0,\dots,n-1\}$.
- We begin by computing the first basis polynomial $L_0$ according to its $n-1$ term definition which still takes $O(n)$ time.
- Then for any $i \in \{1,\dots,n-1\}$ in ${}$, given $L_{i-1}(r)$ we can either compute $L_i(r)$ in constant time or directly evaluate $q_m(r) = m_{r+1}$ for $r \in \{0,\dots,n-1\}$.
  - If $r$ is an element of the interpolating set, the interpolating polynomial simply returns the entry of the message that was paired with $r$, $m_{r+1}$
  - If $r$ is not an element of the interpolating set, the polynomial gives us the $i$th Lagrange polynomial evaluated at $r$
    $$  
    L_i(r) = \frac{L_{i-1}(r) \cdot (r-(i-1))\cdot(-(n-i))}{(r-i)\cdot i}  
    $$
- Since the Lagrange basis polynomials are a constant time calculation with this algorithm, the full evaluation of $q_m(r)$ is a sum of $n$ constant time terms, so the runtime is reduced from $O(n^2)$ to O(n).

```ts
  
/**  
 * Evaluates a univariate Lagrange interpolating polynomial derived from an ASCII message at a specified field element. 
 * 
 * This function interprets the provided ASCII message as function values at distinct interpolation points (indices). 
 * It constructs the Lagrange interpolating polynomial: 
 *   P(x) = Σ (cᵢ · Lᵢ(x))  
 * where each coefficient cᵢ corresponds to the ASCII code of the i-th character in the message,  
 * and Lᵢ(x) is the i-th Lagrange basis polynomial defined over the set {0, 1, ..., n-1}.  
 * 
 * If the evaluation point `r` is one of the interpolation points (i.e., an integer index within the message length), 
 * the function returns the corresponding ASCII value directly for efficiency. 
 * Otherwise, it computes the polynomial value at `r` using Lagrange interpolation. 
 * 
 * @param message - The ASCII string message to encode.  
 * @param r - The field element at which to evaluate the polynomial.  
 * @returns The field element P(r), representing the evaluated polynomial at `r`.  
 * @throws Error if the message is empty or if `r` is an interpolation point when not handled explicitly.  
 */
function getUnivariateLagrange(message: string, r: Field): Field {  
  const n = message.length;  
  if (n === 0) {  
    throw Error("Message cannot be empty.");  
  }  
  // r value is an element in the interpolating set, return the corresponding message entry  
  if (r.lessThan(n).toBoolean()) {  
    const index = Number(r.toBigInt());  
    return Field(asciiToNumber(message[index]));  
  }  
  
  // accumulator for summation over message entries  
  let result = Field(0);  
  // interpolation points ({0...n-1})  
  const interpolatingSet = Array.from({ length: n }, (_, index) =>  
    Field(index),  
  );  
  
  // Initialize 0th Lagrange basis polynomial at r  
  let currentLagrangeBasis = getLagrangeBasisAt(0, interpolatingSet, r);  
  
  // sum over entries in the message multiplied by their corresponding Lagrange basis evaluations  
  for (let i = 0; i < n; i++) {  
    // obtain field representation of ascii character in message  
    const coefficient = Field(asciiToNumber(message[i]));  
  
    // Add the current term (coefficient * current basis) to the result  
    result = result.add(coefficient.mul(currentLagrangeBasis));  
    // Compute the next Lagrange basis polynomial Lᵢ₊₁(r) if not the last term  
  
    if (i < n - 1)  
      currentLagrangeBasis = getNextLagrangeBasisPolynomialAtInput(  
        i + 1,  
        n,  
        r,  
        currentLagrangeBasis,  
      );  
  }  
  
  return result;  
}  
  
  
  
/**  
 * Recursively computes the next Lagrange basis polynomial Lᵢ(r) based on the previous one Lᵢ₋₁(r).  
 * 
 * The recursive formula implemented is: 
 *   Lᵢ(r) = Lᵢ₋₁(r) · (r - (i - 1)) · (-1 * (n - i)) / ((r - i) * i)  
 * 
 * This method assumes that the interpolation set consists of consecutive integers {0, 1, ..., n-1}, * and that `r` is not an element of this set. 
 * 
 * @param i - The index of the Lagrange basis polynomial to compute (must be ≥ 1).  
 * @param n - The size of the interpolating set.  
 * @param r - The field element at which to evaluate the Lagrange basis polynomial.  
 * @param previousLagrangeBasis - The value of the (i-1)-th Lagrange basis polynomial at `r`.  
 * @returns The evaluated value of the i-th Lagrange basis polynomial at `r`.  
 * @throws Error if `i` is less than 1 or if `r` is within the interpolating set.  
 */
function getNextLagrangeBasisPolynomialAtInput(  
  i: number,  
  n: number,  
  r: Field,  
  previousLagrangeBasis: Field,  
) {  
  if (i < 1)  
    throw Error(  
      "Fast Lagrange evaluation algorithm requires the (i-1)th Lagrange basis value at r to calculate the ith lagrange basis value at r",  
    );  
  if (  
    r.greaterThanOrEqual(0).toBoolean() &&  
    r.lessThanOrEqual(n - 1).toBoolean()  
  )  
    throw Error(  
      "Fast Lagrange evaluation algorithm requires that r is not an element of the interpolating set",  
    );  
  
  return previousLagrangeBasis  
    .mul(r.sub(i - 1))  
    .mul(-1 * (n - i))  
    .div(r.sub(i))  
    .div(i);  
}  
  
/**  
 * Computes the i-th Lagrange basis polynomial Lᵢ(r) at a specific field element `r`.  
 * 
 * The Lagrange basis polynomial Lᵢ(x) is defined as:  
 *   Lᵢ(x) = Π₍ⱼ≠i₎ (x - xⱼ) / (xᵢ - xⱼ)  
 * where {x₀, x₁, ..., xₙ₋₁} is the set of interpolation points.  
 * 
 * For the interpolating set consisting of consecutive integers {0, 1, ..., n-1}, 
 * this function evaluates Lᵢ(r) by iterating through each interpolation point except `xᵢ`.  
 * 
 * @param i - The index of the Lagrange basis polynomial to evaluate (0 ≤ i < n).  
 * @param interpolatingSet - The array of interpolation points (must be distinct).  
 * @param r - The field element at which to evaluate the Lagrange basis polynomial.  
 * @returns The evaluated value of the i-th Lagrange basis polynomial at `r`.  
 */
function getLagrangeBasisAt(  
  i: number,  
  interpolatingSet: Field[],  
  r: Field,  
): Field {  
  let result = Field(1);  
  // lagrange basis polynomial is a product over each term in the interpolating set  
  for (let j = 0; j < interpolatingSet.length; j++) {  
    if (i !== j)  
      result = result  
        .mul(r.sub(interpolatingSet[j]))  
        .div(interpolatingSet[i].sub(interpolatingSet[j]));  
  }  
  return result;  
}
```
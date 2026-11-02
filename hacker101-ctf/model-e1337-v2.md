# Hacker101 CTF Write-up: Model E1337 v2 – Hardened Rolling Code Lock

## Challenge Information

| Field | Details |
|---|---|
| Platform | Hacker101 CTF |
| Challenge | Model E1337 v2 – Hardened Rolling Code Lock |
| Category | Cryptography / Reverse Engineering |
| Main technique | PRNG State Recovery |
| Language | Python 3 |

## 1. Introduction

Model E1337 v2 is a rolling-code lock challenge that requires a valid 64-bit code to unlock a web application.

Unlike the original Model E1337 challenge, this version uses a 64-bit internal state and performs three state transitions for every generated output bit.

The objective was to analyze the generator, recover its internal state from an observed output, predict the next rolling code, and use that code to unlock the application.

## 2. Reconnaissance

The homepage presented a simple form requesting an unlock code.

Submitting an incorrect code returned an error message containing the expected rolling code.

This was an important discovery because the application disclosed information about its internal pseudorandom number generator through its error response.

I also discovered that the server exposed the Python generator source code at:

```text
/rng
```

The source revealed two important functions:

- `setup(seed)` initializes the internal state.
- `next(bits)` generates pseudorandom output.

## 3. Understanding the PRNG

The generator uses a 64-bit internal state.

Its output-generation logic can be summarized as:

```python
ret = 0

for _ in range(64):
    ret <<= 1
    ret |= state & 1

    for _ in range(3):
        # Update internal state
```

The important observations were:

1. Each output contains 64 bits.
2. The internal state is updated three times for every output bit.
3. The state transitions are deterministic.
4. The transitions can be represented using matrices over GF(2).

In GF(2), addition is XOR and multiplication is AND.

This mathematical property allows the state recovery problem to be expressed as a system of linear equations.

## 4. State Recovery Method

Rather than brute-forcing the 64-bit code, I used the mathematical structure of the PRNG.

The recovery process follows the technique described in the challenge reference.

### Step 1: Construct the transition matrix

I reproduced the generator's state transition function in Python.

A 64 × 64 transition matrix was constructed to represent three consecutive state updates.

### Step 2: Construct the output equations

The recovery matrix contains 64 equations:

- The first 16 equations represent constraints introduced by the initialization function.
- The remaining 48 equations represent the most significant bits of the observed output.

### Step 3: Gaussian elimination

I implemented Gaussian elimination over GF(2) to solve the resulting system and reconstruct the internal state.

### Step 4: Verify the recovered state

Before predicting another code, I generated an output locally and compared it with the observed challenge response.

The verification succeeded:

```text
Calculated third code: 11197407080074724095
Observed third code:   11197407080074724095

Third code verified.
```

This confirmed that the recovered internal state matched the observed output.

## 5. Complete Python Solver

The following is the complete Python 3 implementation used for the recovery.

Save it as `solve.py`.

```python
MASK = 0xFFFFFFFFFFFFFFFF

# Third expected code captured from the challenge.
OBSERVED = 11197407080074724095


def step(state):
    """One internal state transition from rng.py."""
    state = ((state << 1) ^ (state >> 61)) & MASK
    state ^= MASK

    for j in range(0, 64, 4):
        cur = (state >> j) & 0xF
        cur = (
            (cur >> 3)
            | ((cur >> 2) & 2)
            | ((cur << 3) & 8)
            | ((cur << 2) & 4)
        )
        state ^= cur << j

    return state & MASK


def step_three(state):
    """Perform the three transitions per output bit."""
    for _ in range(3):
        state = step(state)

    return state


def generate(state):
    """Equivalent to next(64) in rng.py."""
    result = 0

    for _ in range(64):
        result = (result << 1) | (state & 1)
        state = step_three(state)

    return result, state


def multiply_matrix(a, b):
    """Matrix multiplication over GF(2)."""
    n = len(a)
    result = [[0] * n for _ in range(n)]

    for i in range(n):
        for k in range(n):
            if a[i][k]:
                for j in range(n):
                    result[i][j] ^= b[k][j]

    return result


def build_transition_matrix():
    """Build the matrix for three internal state transitions."""
    matrix = [[0] * 64 for _ in range(64)]

    for col in range(64):
        basis = 1 << (63 - col)
        output = step_three(basis)

        for row in range(64):
            matrix[row][col] = (
                output >> (63 - row)
            ) & 1

    return matrix


def build_ret_matrix():
    transition = build_transition_matrix()

    ret_matrix = []

    # The 16 initialization constraints.
    for i in range(16):
        bits = f'{9 << (4 * i):064b}'
        ret_matrix.append([
            int(bit) for bit in bits
        ])

    # The 48 output-bit equations.
    state_matrix = [
        [int(i == j) for j in range(64)]
        for i in range(64)
    ]

    for _ in range(48):
        ret_matrix.append(state_matrix[-1][:])

        state_matrix = multiply_matrix(
            state_matrix, transition
        )

    return ret_matrix


def gauss_elim(matrix, vector):
    """Solve a 64x64 system over GF(2)."""
    rows = [
        matrix[i][:] + [vector[i]]
        for i in range(64)
    ]

    pivot = 0

    for col in range(64):
        found = None

        for r in range(pivot, 64):
            if rows[r][col]:
                found = r
                break

        if found is None:
            continue

        rows[pivot], rows[found] = (
            rows[found], rows[pivot]
        )

        for r in range(64):
            if r != pivot and rows[r][col]:
                rows[r] = [
                    x ^ y
                    for x, y in zip(
                        rows[r], rows[pivot]
                    )
                ]

        pivot += 1

    if pivot != 64:
        raise ValueError(
            f"Matrix rank is {pivot}, expected 64."
        )

    for row in rows:
        if not any(row[:64]) and row[64]:
            raise ValueError("Inconsistent equations.")

    return [rows[i][64] for i in range(64)]


def main():
    print("[*] Building transition matrix...")
    ret_matrix = build_ret_matrix()

    print("[*] Preparing observed code...")
    codes = ('0' * 16 + f'{OBSERVED:064b}')[:64]

    codes_vector = [
        int(bit) for bit in codes
    ]

    print("[*] Recovering internal state...")
    recovered = gauss_elim(
        ret_matrix, codes_vector
    )

    state = int(''.join(map(str, recovered)), 2)

    print("[+] Internal state recovered.")

    # Verify the recovered state.
    calculated, state = generate(state)

    print("[*] Calculated third code:", calculated)
    print("[*] Observed third code:  ", OBSERVED)

    if calculated != OBSERVED:
        raise SystemExit(
            "[-] Verification failed. Do not submit a code."
        )

    print("[+] Third code verified.")

    # Calculate the next rolling code.
    next_code, state = generate(state)

    print("[+] NEXT EXPECTED CODE:", next_code)
    print("[+] HEX:", hex(next_code))


if __name__ == "__main__":
    main()
```

### Running the solver

```bash
python3 solve.py
```

### Verified output

```text
[*] Building transition matrix...
[*] Preparing observed code...
[*] Recovering internal state...
[+] Internal state recovered.
[*] Calculated third code: 11197407080074724095
[*] Observed third code:   11197407080074724095
[+] Third code verified.
[+] NEXT EXPECTED CODE: 17777770138705342182
[+] HEX: 0xf6b753c020142ee6
```

## 6. Exploitation

The recovered state allowed me to predict the next rolling code:

```text
17777770138705342182
```

The code can be submitted to the unlock endpoint using:

```bash
curl -ki -X POST "$TARGET/unlock" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data "code=17777770138705342182"
```

**Flag:**

```text
^FLAG^[REDACTED]$FLAG$
```

## 7. Security Impact

This challenge demonstrates several security weaknesses:

- **Predictable PRNG:** Deterministic state transitions make future outputs recoverable.
- **Information disclosure:** Error messages reveal expected authentication codes.
- **Weak authentication design:** Rolling codes become ineffective when their underlying generator can be reverse-engineered.
- **Source code exposure:** Exposing the generator implementation makes analysis easier.

## 8. Recommended Mitigations

For real-world applications:

1. Use cryptographically secure random number generators.
2. Never disclose expected authentication codes in error messages.
3. Use established cryptographic constructions for rolling-code systems.
4. Implement replay protection and code expiration.
5. Add rate limiting and abuse detection to authentication endpoints.
6. Avoid relying on custom PRNG implementations for security.

## 9. Key Takeaways

This challenge was a practical exercise in reverse engineering, cryptanalysis, and Python automation.

The biggest lesson was that a large output space does not automatically make a system secure. When a pseudorandom number generator has predictable internal state transitions, mathematical techniques such as matrix construction and Gaussian elimination can be used to recover its state and predict future outputs.

**Techniques practiced:**

- Source code analysis
- PRNG reverse engineering
- 64-bit state recovery
- Matrix construction and multiplication
- Gaussian elimination over GF(2)
- Python automation

---

*Completed as part of my Hacker101 CTF practice.*

*Flags have been redacted for responsible public sharing.*
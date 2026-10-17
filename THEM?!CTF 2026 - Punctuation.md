# Punctuation.?! - Writeup Written May 31, 2026

**100 points | Cryptography**

This challenge uses the **Franklin-Reiter attack against RSA**. If two messages differ only by a known fixed difference and are RSA encrypted under the same RSA modulus `N`, then it is possible to recover both of them.

## Step 1: Calculate the Difference

First, take the given information and calculate the difference between the two messages.

```python
m_question = bytes_to_long(b"THEM?" + FLAG + b"!")
m_bang = bytes_to_long(b"THEM!" + FLAG + b"?")
```

A byte ranges from 0 to 255, so when the original script converts these to integers, the base is 256.

This means that to calculate the difference between the two numbers, we calculate the difference between the swapped characters (`?` and `!` at the position before the flag, and `!` and `?` at the end).

We are given that the flag length is 43 bytes. As such, we can solve for the difference:

```python
r = (ord('?') - ord('!')) * (256 ** 44) + ord('!') - ord('?')
```

## Step 2: Implement the Attack

At this point, we can implement the attack. I used a SageMath implementation of the Franklin–Reiter attack adapted from ValarDragon's CTF-Crypto repository.

The complete solve script is below:

[https://github.com/ValarDragon/CTF-Crypto/blob/master/RSA/FranklinReiter.sage](https://github.com/ValarDragon/CTF-Crypto/blob/master/RSA/FranklinReiter.sage)

Here is the full solve:

```python
from Crypto.Util.number import long_to_bytes

# r is the known difference
# Ciphertext 1 corresponds to the smaller plaintext
# (the one without the fixed difference added to it)
def franklinReiter(n, e, r, c1, c2):
    R.<X> = Zmod(n)[]
    f1 = X^e - c1
    f2 = (X + r)^e - c2

    # Coefficient 0 = -m, which is what we want
    return Integer(n - (compositeModulusGCD(f1, f2)).coefficients()[0])


# GCD is not implemented for rings over composite modulus in Sage,
# so we use our own implementation with monic polynomials
def compositeModulusGCD(a, b):
    if b == 0:
        return a.monic()
    else:
        return compositeModulusGCD(b, a % b)


def attackFranklinReiter():
    # Given information
    n = 88183024196256411333484553213553846837622439231780396480967109312314357913174425385396184231905765939381960367698537125376473533288141531434345042789797178645864135392661922564207259941331338650097600270616742579575453179095821632880008651838424568933498141806505982361171994397688679468138932151808009068179

    e = 3

    c_bang = 66665542371335779429364935032331984667182055425006583461719436282420129123847028566349996921560146372476231977506705605038813943625646058608138540817207845848101241000676402022548634430605544136846901869868655052580588798913771078650808191399486846694919669525753981881703314216258160513542942415095706258060

    c_question = 43862735413007212838921290204740237228920015804978660936092290351869586236148422881135933115705865580189043164227764444579442212567444887615221721393945266948407026475728221628198734395544594052170760423327865044863524666382485613853620502506826243391273684653499522966102384658131639487384266064123758554996

    r = (ord('?') - ord('!')) * (256 ** 44) + ord('!') - ord('?')

    recoveredM = franklinReiter(n, e, r, c_bang, c_question)
    recoveredM = long_to_bytes(recoveredM).decode()

    print(recoveredM)


attackFranklinReiter()
```

## Explanation

In very simple terms, the two messages are basically the same message with the punctuation swapped. Because we can calculate the difference, one message is just the other message plus a known difference `r`.

Since both were encrypted with the same RSA modulus `n` and `e = 3`, the two ciphertexts create related equations. The attack finds the shared solution to those equations, which gives us the original plaintext.

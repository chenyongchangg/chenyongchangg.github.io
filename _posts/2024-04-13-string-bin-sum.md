## Question
Given two binary strings a and b, return their sum as a binary string.



Example 1:

Input: a = "11", b = "1"
Output: "100"
Example 2:

Input: a = "1010", b = "1011"
Output: "10101"


Constraints:

1 <= a.length, b.length <= 104
a and b consist only of '0' or '1' characters.
Each string does not contain leading zeros except for the zero itself.

## Code Answer
```Python
class Solution:
    def addBinary(self, a: str, b: str) -> str:
        r, p = '', 0
        diff = len(a) - len(b)
        a = '0' * -diff + a
        b = '0' * diff + b
        for i, j in zip(a[::-1], b[::-1]):
            su = int(i) + int(j) + p
            r = str(su % 2) + r
            p = su // 2
        return '1' + r if p > 0 else r

```

## Explain

* reverse arr
* use zip
* use return r if .. else ,,
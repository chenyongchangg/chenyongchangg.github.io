# just see example
```Python
from collections import *

if __name__ == '__main__':
    print([[2] * 10])
    # [[2, 2, 2, 2, 2, 2, 2, 2, 2, 2]]

    print([[2] for i in range(10)])
    # [[2], [2], [2], [2], [2], [2], [2], [2], [2], [2]]

    dic = defaultdict(list)
    dic[1].append(3)
    dic["sdf"].append(3)
    print(dic)
    # {1: [3], 'sdf': [3]})

    

```
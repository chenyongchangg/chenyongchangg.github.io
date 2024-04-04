# python useful code
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

    s = set()
    s.add(1)
    s.add("sdf")
    print(s)
    # {1, 'sdf'}

    int_list = [2]
    int_list.append([3, 2])
    print(int_list)
    # [2, [3, 2]]
    int_list += [3, 2]
    print(int_list)
    # [2, [3, 2], 3, 2]

    # 下面这个初始化方式可以的 ，都是小写
    join_tab = defaultdict(list)
    anc = defaultdict(set)



    

    

```
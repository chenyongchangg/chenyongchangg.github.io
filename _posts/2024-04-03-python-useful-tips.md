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

## heapq
heapq不太好用，不能在heapify中传入lamda，只能在push的时候传入lamda，只能是小顶堆
```class Solution:
    def trapRainWater(self, heightMap: List[List[int]]) -> int:
        if len(heightMap) <= 2 or len(heightMap[0]) <= 2:
            return 0
        my_heap = []
        res = 0
        row_num = len(heightMap)
        col_num = len(heightMap[0])
        vis = [[False for _ in range(col_num)] for _ in range(row_num)]
        for i in range(row_num):
            heapq.heappush(my_heap, [heightMap[i][0], i, 0])
            heapq.heappush(my_heap, [heightMap[i][col_num-1], i, col_num-1])
            vis[i][0] = True
            vis[i][col_num-1] = True
        for j in range(1, col_num-1, 1):
            heapq.heappush(my_heap, [heightMap[0][j], 0, j])
            heapq.heappush(my_heap, [heightMap[row_num-1][j], row_num-1, j])
            vis[0][j] = True
            vis[row_num-1][j]  = True
        cur_hei = 0
        while len(my_heap) > 0:
            cur_node = heapq.heappop(my_heap)
            cur_hei = max(cur_hei, cur_node[0])
            for [i, j] in [[1, 0], [-1, 0], [0, 1], [0, -1]]:
                ii = cur_node[1] + i
                jj = cur_node[2] + j
                if  1 <= ii <= row_num-2 and 1 <= jj <= col_num -2 and (not vis[ii][jj]):
                    ii = cur_node[1] + i
                    jj = cur_node[2] + j
                    res += max(0, (cur_hei-heightMap[ii][jj]))
                    heapq.heappush(my_heap, [heightMap[ii][jj], ii ,jj])
                    vis[ii][jj] = True
        return res
                    



```
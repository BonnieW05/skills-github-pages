---
layout: post
title: "倒水问题的几种解法"
date: 2025-03-03
---

前几天上人智导，老师回顾到搜索算法时提出了一个问题：设有三个没有刻度的杯子，分别可以装8两、8两和3两水。两个8两的杯子装满了水，请问如何在不借助于其他器具的情况下，让4个人每人喝到4两水。
课堂上gpt和deepseek都没能很好地解答这个问题，分别出现了认为水能倒出一半和直接倒到溢出的问题。后来下课后我试图让它们用编程解决该问题，也以失败告终。
当我试图寻找编程解法时，在网络上并没有找到相关程序，于是便自己写了一下。虽然用了很长时间（唉还是菜了），代码写的也相当丑陋，但是还是想分享一下。
（如果想直接查看答案的话请翻到文末）
版本一：
```cpp
#include <iostream>
#include <queue>
#include <vector>
#include <unordered_map>
using namespace std;

class state {
public:
    int data[7]; // 前3个表示杯子的水量，后4个表示4个人的水量

    state() {
        for (int i = 0; i < 7; ++i)
            data[i] = 0;
    }

    state(int a, int b, int c, int d, int e, int f, int g) {
        data[0] = a;  // 8两
        data[1] = b;  // 8两
        data[2] = c;  // 3两
        data[3] = d;
        data[4] = e;
        data[5] = f;
        data[6] = g; 
    }

    bool operator==(const state& other) const {
        for (int i = 0; i < 7; ++i) {
            if (data[i] != other.data[i])
                return false;
        }
        return true;
    }

    void print() const {
        cout << "(" << data[0] << ", " << data[1] << ", " << data[2] << ", " 
             << data[3] << ", " << data[4] << ", " << data[5] << ", " << data[6] << ")" << endl;
    }
};

bool isSame(vector<state> v, state s) {
    for (int i = 0; i < v.size(); i++) 
        if (v[i] == s) 
            return true;

    return false;
}

struct stateHash {
    size_t operator()(const state& s) const {
        size_t hashVal = 0;
        for (int i = 0; i < 7; ++i) {
            hashVal ^= std::hash<int>()(s.data[i]) + 0x9e3779b9 + (hashVal << 6) + (hashVal >> 2);
        }
        return hashVal;
    }
};

void printPath(const unordered_map<state, state, stateHash>& parentMap, const state& start, const state& target) {
    vector<state> path;
    state current = target;
    while (!(current == start)) {
        path.push_back(current);
        current = parentMap.at(current);  // 回溯父状态
    }
    path.push_back(start);

    // 反向输出路径
    for (int i = path.size() - 1; i >= 0; --i) {
        path[i].print();
    }
}

int main() {
    state start(8, 8, 0, 0, 0, 0, 0);
    state target(0, 0, 0, 4, 4, 4, 4);

    queue<state> q; // BFS 队列
    unordered_map<state, state, stateHash> parentMap; // 记录每个状态的父状态
    vector<state> visited; // 记录已访问的状态

    q.push(start);
    visited.push_back(start);

    while (!q.empty()) {
        state current = q.front();
        q.pop();

        if (current == target) {
            cout << "Found solution!" << endl;
            printPath(parentMap, start, current);
            cout<<endl<<endl;
        }

        // 1. 倒水给人（杯子到人）
        for (int i = 0; i < 3; ++i) {
            for (int j = 3; j < 7; ++j) {
                if (current.data[i] > 0 && current.data[i] + current.data[j] <= 4) {
                    state next = current;

                    int transferAmount = min(next.data[i], 4 - next.data[j]);

                    next.data[i] -= transferAmount;
                    next.data[j] += transferAmount;

                    if (isSame(visited, next) == false) {
                        visited.push_back(next);
                        q.push(next);
                        parentMap[next] = current;
                    }
                }
            }
        }

        // 2. 杯子之间倒水（杯子到杯子）
        for (int i = 0; i < 3; ++i) {
            for (int j = 0; j < 3; ++j) {
                if (i != j && current.data[i] > 0 && current.data[j] < ((j == 0 || j == 1) ? 8 : 3)) {
                    state next = current;

                    int transferAmount = min(next.data[i], ((j == 0 || j == 1) ? 8 : 3) - next.data[j]);

                    next.data[i] -= transferAmount;
                    next.data[j] += transferAmount;

                    if (isSame(visited, next) == false) {
                        visited.push_back(next);
                        q.push(next);
                        parentMap[next] = current;
                    }
                }
            }
        }
    }

    cout << "No solution found." << endl;
    return 0;
}
```
其中打印路径的函数我通过询问gpt得到。
另外我觉得似乎有一种更优美的写法：直接将状态作为七位int进行存储，这样就不用新定义一个类，而自动拥有比较，判等等性质，而且可以写一个函数来提取第n位，写一个函数来改变第n位（似乎也不是很难）。可能这样写会更好吧。
（一更）现已经修改，修改后的程序如下（版本二）：
```cpp
#include <iostream>
#include <queue>
#include <unordered_map>
using namespace std;

typedef int State;

int get(State s, int n) {
    for (int i = 0; i < n; ++i) s /= 10;
    return s % 10;
}

// 设置第 n 位的值
State set(State s, int n, int val) {
    State factor = 1;
    for (int i = 0; i < n; ++i) factor *= 10;
    return s - get(s, n) * factor + val * factor;
}

// 打印状态
void printState(State s) {
    for (int i = 0; i < 7; ++i) 
        cout << get(s, i) << (i < 6 ? ", " : "\n");
}

// 还原路径
void printPath(unordered_map<State, State>& parent, State start, State target) {
    vector<State> path;
    while (target != start) {
        path.push_back(target);
        target = parent[target];
    }
    path.push_back(start);
    for (int i = path.size() - 1; i >= 0; --i) {
        printState(path[i]);
    }
}

int main() {
    State start = 88; // (0,0,0,0,0,8,8)
    State target = 4444000; // (4,4,4,4,0,0,0)
    
    queue<State> q;
    unordered_map<State, State> parent;
    unordered_map<State, bool> visited;

    q.push(start);
    visited[start] = true;

    while (!q.empty()) {
        State current = q.front();
        q.pop();
        if (current == target) {
            cout << "Found solution!\n";
            printPath(parent, start, current);
            cout << endl << endl;
            continue;
        }

        // 杯子倒水给人
        for (int i = 0; i < 3; ++i) {
            for (int j = 3; j < 7; ++j) {
                int ci = get(current, i), cj = get(current, j);
                if (ci > 0 && cj + ci <= 4) {
                    int transfer = min(ci, 4 - cj);
                    State next = set(set(current, i, ci - transfer), j, cj + transfer);
                    if (!visited[next]) {
                        visited[next] = true;
                        q.push(next);
                        parent[next] = current;
                    }
                }
            }
        }

        // 杯子之间倒水
        for (int i = 0; i < 3; ++i) {
            for (int j = 0; j < 3; ++j) {
                if (i != j) {
                    int ci = get(current, i), cj = get(current, j);
                    int limit = (j < 2) ? 8 : 3;
                    if (ci > 0 && cj < limit) {
                        int transfer = min(ci, limit - cj);
                        State next = set(set(current, i, ci - transfer), j, cj + transfer);
                        if (!visited[next]) {
                            visited[next] = true;
                            q.push(next);
                            parent[next] = current;
                        }
                    }
                }
            }
        }
    }

    //cout << "No solution found." << endl;
    return 0;
}
```

浅挖个坑：今天课上学了A算法，我也许还试着用该算法写一版并且比较一下性能。
（二更）写了。我记录了一下需要出栈入栈的次数，发现第一次找到解（步数一致）广搜和A用的步数分别为6013和5332，感觉优势不是很明显（
我的估计函数思路是取max（没倒完的罐，没喝饱的人），也许还会有更好的。
版本三：
```cpp
#include <iostream>
#include <queue>
#include <unordered_map>
#include <cmath>
using namespace std;

int op_count = 0;

// 最少也还要倒的次数
int heuristic(int state) {
    int low, high;
    for (int i = 0; i < 3; ++i)
        if (state / (int)pow(10, i) % 10 > 0) low++;
    for (int i = 3; i < 7; ++i)
        if (state / (int)pow(10, i) % 10 < 4) high++;
    return max(high, low);
}

int get(int s, int n) {
    for (int i = 0; i < n; ++i) s /= 10;
    return s % 10;
}

// 设置某一位的值（n 从 0 开始）
int set(int s, int n, int val) {
    int factor = 1;
    for (int i = 0; i < n; ++i) factor *= 10;
    return s - get(s, n) * factor + val * factor;
}

void printState(int s) {
    for (int i = 0; i < 7; ++i) 
        cout << get(s, i) << (i < 6 ? ( i == 2 ? " | " : " , ") : "\n");
}

void printPath(const unordered_map<int, int>& parent, int start, int target) {
    vector<int> path;
    int current = target;
    int count = 0;
    while (current != start) {
        path.push_back(current);
        current = parent.at(current);
    }
    path.push_back(start);
    
    for (int i = path.size() - 1; i >= 0; --i){
        printState(path[i]);
        count++;
    }
    cout << "Total steps: " << count << endl;
}

int main() {
    int start = 88;
    int target = 4444000;
    bool flag = false;
    
    priority_queue<pair<int, int>, vector<pair<int, int>>, greater<pair<int, int>>> pq; //前数是预计前路代价，后数是状态
    unordered_map<int, int> parent;
    unordered_map<int, int> cost; //到此最小代价
    
    pq.push({heuristic(start), start});
    cost[start] = 0;
    
    while (!pq.empty()) {
        int current = pq.top().second;
        pq.pop();
        op_count++;
        
        if (current == target) {
            cout << "Found solution!" << endl;
            printPath(parent, start, current);
            flag = true;
            break;
            //continue;
        }
        
        // 杯子倒水给人
        for (int i = 0; i < 3; ++i) {
            for (int j = 3; j < 7; ++j) {
                int ci = get(current, i), cj = get(current, j);
                if (ci > 0 && cj + ci <= 4) {
                    int transfer = min(ci, 4 - cj);
                    int next = set(set(current, i, ci - transfer), j, cj + transfer);
                    if (!cost.count(next) || cost[next] > cost[current] + 1) {
                        cost[next] = cost[current] + 1;
                        pq.push({cost[next] + heuristic(next), next});
                        parent[next] = current;
                    }
                }
            }
        }
        
        // 杯子之间倒水
        for (int i = 0; i < 3; ++i) {
            for (int j = 0; j < 3; ++j) {
                if (i != j) {
                    int ci = get(current, i), cj = get(current, j);
                    int limit = (j < 2) ? 8 : 3;
                    if (ci > 0 && cj < limit) {
                        int transfer = min(ci, limit - cj);
                        int next = set(set(current, i, ci - transfer), j, cj + transfer);
                        if (!cost.count(next) || cost[next] > cost[current] + 1) {
                            cost[next] = cost[current] + 1;
                            pq.push({cost[next] + heuristic(next), next});
                            parent[next] = current;
                        }
                    }
                }
            }
        }
    }
    
    if (!flag) cout << "No solution found." << endl;
    cout << "Total operations: " << op_count << endl;
    return 0;
}
```

最后得到的（一种）答案是：
(8, 8, 0, 0, 0, 0, 0)
(5, 8, 3, 0, 0, 0, 0)
(5, 8, 0, 3, 0, 0, 0)
(2, 8, 3, 3, 0, 0, 0)
(0, 8, 3, 3, 2, 0, 0)
(3, 8, 0, 3, 2, 0, 0)
(3, 5, 3, 3, 2, 0, 0)
(6, 5, 0, 3, 2, 0, 0)
(6, 2, 3, 3, 2, 0, 0)
(8, 2, 1, 3, 2, 0, 0)
(8, 2, 0, 4, 2, 0, 0)
(5, 2, 3, 4, 2, 0, 0)
(0, 7, 3, 4, 2, 0, 0)
(3, 7, 0, 4, 2, 0, 0)
(3, 4, 3, 4, 2, 0, 0)
(6, 4, 0, 4, 2, 0, 0)
(6, 1, 3, 4, 2, 0, 0)
(6, 0, 3, 4, 2, 1, 0)
(8, 0, 1, 4, 2, 1, 0)
(8, 0, 0, 4, 2, 1, 1)
(5, 0, 3, 4, 2, 1, 1)
(5, 0, 0, 4, 2, 4, 1)
(2, 0, 3, 4, 2, 4, 1)
(0, 0, 3, 4, 4, 4, 1)
(0, 0, 0, 4, 4, 4, 4)

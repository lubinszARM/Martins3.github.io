https://www.zhihu.com/question/399923003/answer/1272134898 : 有一个学习路线

资源:
https://github.com/geektutu/7days-golang : 如果真的想搞分布式，那么这就是最佳的入手
https://github.com/inancgumus/learngo : 通过例子学习 golang

https://github.com/shomali11/go-interview : go 语言基础
https://github.com/go-vgo/robotgo : 应该很有意思的东西，用于实现桌面自动化

https://github.com/golang-standards/project-layout : go 项目安排，c++ 有类似的教学 repo

https://github.com/Alikhll/golang-developer-roadmap : golang 的 roadmap，看来都是搞后端的

# Go
The reason to learn go is that I don't want to miss a good community!
As for how to learn it, I think the best !

:= a shothand for create a variable and assign a value to it !





## Interface
It seems that go is going back to the C. No class, only struct.

Interface heavily rely on **type**, which means T * and T are totally different !

Interface just work with method receiver ?

Interfaces are implemented implicitly
A type implements an interface by implementing its methods. There is no explicit declaration of intent, no "implements" keyword.
Implicit interfaces decouple the definition of an interface from its implementation, which could then appear in any package without prearrangement.




## tips
1. [slice](https://stackoverflow.com/questions/39984957/is-it-possible-to-initialize-golang-slice-with-specific-values)
```
onesSlice := make([]int, 5)
for i := range onesSlice {
    onesSlice[i] = 1
}
```
2. 

## Question
1. what's make, is a **new** in c++
2. is there generic 
3. does go has copy-constructor
4. code as below
```
type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *IntHeap) Push(x interface{}) {
    // Push and Pop use pointer receivers because they modify the slice's length,
    // not just its contents.
    *h = append(*h, x.(int))
}

func (h *IntHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[0 : n-1]
    return x
}
```
why return type is `interface{}`


## External Links
1. https://tour.go-zh.org/basics/7
2. https://github.com/go101/go101/issues/132
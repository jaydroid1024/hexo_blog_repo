---
title: 筑基系列-算法大全（更新中...）
date: 2021-10-31 14:16:55
cover: true
tags: 

    - 算法
category: 
	- 算法
summary: 链表、排序、字符串、二叉树等等

---



<details>    
  <summary> <b> Solution2：两次循环+栈，时间：O(N),空间：O(N) </b></summary>      
   <pre><code>
	代码
  </code></pre>
</details>


### 01 组中重复的数字

| 剑指 Offer 03 | [数组中重复的数字](https://leetcode-cn.com/problems/shu-zu-zhong-zhong-fu-de-shu-zi-lcof/) | [LeetCode 题解](https://leetcode-cn.com/problems/shu-zu-zhong-zhong-fu-de-shu-zi-lcof/solution/) |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |





### 01 从尾到头打印链表

| 剑指 Offer 06 | [从尾到头打印链表](https://leetcode-cn.com/problems/cong-wei-dao-tou-da-yin-lian-biao-lcof) | [LeetCode 题解](https://leetcode-cn.com/problems/cong-wei-dao-tou-da-yin-lian-biao-lcof/solution) |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |

<details>    
  <summary> <b> Solution1：两次循环，时间：O(N)，空间：O(N)</b> </summary>      
   <pre><code>
/*
  Solution1：两次循环，时间：O(N),空间：O(N)
  1. while 循环遍历链表求出总长度 length
  2. 根据length构建整数型数组 array
  3. for 循环向 array 中后插法插入数据
 */
class Solution1 {
    fun reversePrint(head: ListNode?): IntArray {
        var temp = head
        var length = 0
        //1. while 循环遍历链表求出总长度 length
        while (temp != null) {
            length++
            temp = temp.next
        }
        //2. 根据length构建整数型数组 array
        val array = IntArray(length)
        temp = head//temp指针指向头节点再利用
        //3. for 循环向 array 中后插法插入数据
        for (i in 0 until length) {
            array[length - 1 - i] = temp?.`val` ?: 0
            temp = temp?.next
        }
        return array
    }
}
  </code></pre>
</details>
<details>    
  <summary> <b> Solution2：两次循环+栈，时间：O(N),空间：O(N) </b></summary>      
   <pre><code>
/*
    Solution2：两次循环+栈，时间：O(N),空间：O(N)
    1. 入栈： 遍历链表，将各节点值 addLast 入栈。（借助 LinkedList 的addLast()方法）。
    2. 出栈： 将各节点值 removeLast 出栈，存储于数组并返回。
 */
class Solution2 {
    fun reversePrint(head: ListNode?): IntArray {
        //stack.push(temp) / stack.pop() ，Java 中的Stack数据结构是继承自Vector，addElement方法是通过大锁保证线程安全的
        //Stack<ListNode> stack = new Stack<>();
        //通过 LinkedList 来模拟栈操作可以提高效率
        var temp = head
        val stack = LinkedList<Int>()
        //1. 入栈： 遍历链表，将各节点值 addLast 入栈。（借助 LinkedList 的addLast()方法）。
        while (temp != null) {
            stack.addLast(temp.`val`)
            temp = temp.next
        }
        val res = IntArray(stack.size)
        //2. 出栈： 将各节点值 removeLast 出栈，存储于数组并返回。
        for (i in res.indices) {
            res[i] = stack.removeLast()
        }
        return res
    }
}
  </code></pre>
</details>
<details>    
  <summary> <b> Solution3：递归，时间：O(N)，空间：O(N) </b></summary>      
   <pre><code>
/*
    Solution3：递归，时间：O(N),空间：O(N)
    1. 递推阶段：每次传入 head.next ，以 head == null（即走过链表尾部节点）为递归终止条件，此时直接返回。
    2. 回溯阶段：层层回溯时，将当前节点值加入列表，即tmp.add(head.val)。
    3. 转换数据：将列表 tmp 转化为数组 res ，并返回即可。
 */
class Solution3 {
    var tmp = ArrayList<Int>()
    fun reversePrint(head: ListNode?): IntArray {
        recur(head)
        //转换数据：将列表 tmp 转化为数组 res ，并返回即可。
        val res = IntArray(tmp.size)
        for (i in res.indices) res[i] = tmp[i]
        return res
    }
    fun recur(head: ListNode?) {
    //1. 递推阶段：每次传入 head.next ，以 head == null（即走过链表尾部节点）为递归终止条件，此时直接返回。
    if (head == null) return
    recur(head.next)
    //2. 回溯阶段：层层回溯时，将当前节点值加入列表，即tmp.add(head.val)。
    tmp.add(head.`val`)
	}
}
//递归2
class Solution31 {
    private lateinit var res: IntArray
    private var i = 0 //测量栈深度，初始化数组
    private var x = 0 //数组下标
    fun reversePrint(head: ListNode?): IntArray {
        solve(head)
        return res
    }
    fun solve(head: ListNode?) {
    //1. 递推阶段：每次传入 head.next ，以 head == null（即走过链表尾部节点）为递归终止条件，此时获取了递归深度可以初始化数组再返回。
    if (head == null) {
        res = IntArray(i)
        return
    }
    i++
    solve(head.next)
    //2. 回溯阶段：层层回溯时，将当前节点值加入列表，即tmp.add(head.val)。
    res[x] = head.`val`
    x++
	}
}
  </code></pre>
</details>









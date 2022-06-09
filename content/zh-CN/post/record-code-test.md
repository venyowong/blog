---
title: 记一次机试过程
date: 2020-02-10 19:20:39
tags:
  - java
  - 最长子串
---

上个礼拜做了一次机试，题目类似与 [最长有效括号](https://leetcode-cn.com/problems/longest-valid-parentheses/) 这道题，但是题目中的 `'(', ')'` 被更换成了 `'(', ')', '[', ']', '{', '}'`，这个修改明显增加了题目的复杂度，但又恰到好处。

首先来说下我一开始的思路：直观地思考这道题，应该是可以通过一次字符串的遍历，来得到最终答案。随着从左到右遍历，遇到的左括号会越来越多，而在遇到右括号之后，待匹配的左括号变少一个，"游标"会向左移一位，这个过程容易让人联想到入栈出栈，因此数据结构选择栈来实现。算法用自然语言描述大致如下：
1. 定义一个栈 stack 用于存储左括号，定义一个 type 记录栈顶左括号的类型，小括号为 1，中括号为 2，大括号为 3，定义长度变量 len，默认值为 0，定义一个长度数组 lens
2. 遍历字符串 input，当前字符为左括号转 3，否则转 4，遍历结束转 7
3. 字符入栈 stack，更新括号类型 type，转 2
4. type = 0 转 2，否则判断括号类型与 type 是否能够匹配(即是否是有效的一对括号)，若匹配转 5，否则转 6
5. stack 出栈，并更新 type 为当前栈顶元素的括号类型，len += 2，转 2
6. lens.add(len)，len = 0，stack.clear()，type = 0，转 2
7. lens.add(len)
8. lens 获取最大值，输出

代码提交之后，发现只过了百分之八十的测试用例，我自己写的几个测试用例也都能过，比如：`((((()[]}}}`，`)))))(]`，`()((()]}([{}])` 等。

由于人工编写测试用例，难免会有遗漏，因此我当时写了一个随机生成测试用例的代码：
```
Random random = new Random(new Date().getTime());
String input = "";
int size = random.nextInt(100);
for (int i = 0; i < size; i++) {
    int r = random.nextInt(6);
    switch (r) {
        case 0:
            input += '(';
            break;
            case 1:
            input += '{';
            break;
            case 2:
            input += '[';
            break;
            case 3:
            input += ')';
            break;
            case 4:
            input += '}';
            break;
            case 5:
            input += ']';
            break;
    }
}
System.out.println(input);
```
大概测了七八次，出现了一个有问题的测试用例，由于有大量无用子串，因此我只列出最关键的部分：`()[{}`，这个输入在使用上面的算法时，得到的答案是 4，但很明显答案是 2。

我使用这个用例调试了一下，发现算法确实存在很大的漏洞：步骤 5 由于累加长度以后，会对左括号做出栈操作，这其实已经破坏了结构，导致某个无法得到匹配的左括号能够把与之相邻的两个左右子串连接起来成为一个"伪装的有效子串"。比如 `()[{}` 该字符串遍历至 `input[2]` 时，`[` 入栈，此时 stack 中只有这一个元素，因为前面的小括号已经匹配成功出栈了，且 len 的数值为 2。忽略第一对小括号，`[{}` 该字符串的有效子串为 2，并且遍历过程中没有走到步骤 6，因此 len 会累加 2，变成 4，于是得到了错误答案。

遗憾的是，到这时，测试时间刚好到了，于是成绩就是百分之八十通过率。但其实修改逻辑漏洞也消耗了我一段时间，所以测试时间稍加延长，也无法让我提交一份完美通过的代码了。当时的自己开始感叹自己毕业了几年，原来只是一直在变笨而已。经过修改，最终的算法如下：
1. 定义一个栈 ls 用于存储左括号，定义一个栈 rs 用于存储右括号，定义一个 type 记录栈顶左括号的类型，小括号为 1，中括号为 2，大括号为 3，定义一个长度数组 lens，定义一个栈 types 用于存储 type
2. 遍历字符串 input，当前字符为左括号转 3，否则转 4，遍历结束转 7
3. 字符入栈 ls，更新括号类型 type，type 入栈 types，转 2
4. 判断括号类型与 type 是否能够匹配(即是否是有效的一对括号)，若匹配转 5，否则转 6
5. 字符入栈 rs，types 栈顶元素出栈，更新 type 为 types 栈顶元素值，转 2
6. 通过 ls 与 rs 获取最长有效子串 len(该过程定义为 getLength，详见代码)，lens.add(len)，ls.clear()，rs.clear()，转 2
7. lens.add(getLength(ls, rs))
8. lens 获取最大值，输出

getLength 代码如下：
```
private static int getLength(Stack<Character> ls, Stack<Character> rs) {
    int len = 0, max = 0;
    while (rs.size() > 0 && ls.size() > 0) {
        char l = ls.pop();
        char r = rs.pop();
        if (getType(l) == getRightType(r)) {
            len += 2;
        }
        else {
            rs.push(r);
            if (len > max) {
                max = len;
            }
            len = 0;
        }
    }
    return max;
}
```
由于后来的修改未提交测试验证过，所以也无法保证一定不存在问题，因此若读者发现了任何问题，希望能及时告诉我，我会第一时间处理，以免这篇文章误导了更多人。邮箱地址：venyowong@163.com

完整代码如下：
```
package javademo;

import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Random;
import java.util.Scanner;
import java.util.Stack;

public class Main {

    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        String input = scanner.nextLine();
        if (input == null || input.isEmpty()) {
            System.out.print(0);
            return;
        }
        
//        Random random = new Random(new Date().getTime());
//        String input = "";
//        int size = random.nextInt(100);
//        for (int i = 0; i < size; i++) {
//            int r = random.nextInt(6);
//            switch (r) {
//                case 0:
//                    input += '(';
//                    break;
//                    case 1:
//                    input += '{';
//                    break;
//                    case 2:
//                    input += '[';
//                    break;
//                    case 3:
//                    input += ')';
//                    break;
//                    case 4:
//                    input += '}';
//                    break;
//                    case 5:
//                    input += ']';
//                    break;
//            }
//        }
//        System.out.println(input);
        
        Stack<Character> ls = new Stack<>();
        Stack<Character> rs = new Stack<>();
        Stack<Integer> types = new Stack<>();
        int type = 0;
        List<Integer> lens = new ArrayList<Integer>();
        for (int i = 0; i < input.length(); i++) {
            char c = input.charAt(i);
            switch (c) {
                case '(':
                    ls.push(c);
                    types.push(1);
                    type = 1;
                    break;
                case '[':
                    ls.push(c);
                    types.push(2);
                    type = 2;
                    break;
                case '{':
                    ls.push(c);
                    types.push(3);
                    type = 3;
                    break;
                case ')':
                    if (type == 1 && ls.size() > 0) {
                        rs.push(c);
                        type = getLastType(types);
                    }
                    else {
                        lens.add(getLength(ls, rs));
                        ls.clear();
                        rs.clear();
                    }
                    break;
                case ']':
                    if (type == 2 && ls.size() > 0) {
                        rs.push(c);
                        type = getLastType(types);
                    }
                    else {
                        lens.add(getLength(ls, rs));
                        ls.clear();
                        rs.clear();
                    }
                    break;
                case '}':
                    if (type == 3 && ls.size() > 0) {
                        rs.push(c);
                        type = getLastType(types);
                    }
                    else {
                        lens.add(getLength(ls, rs));
                        ls.clear();
                        rs.clear();
                    }
                    break;
                default:
                    lens.add(getLength(ls, rs));
                    ls.clear();
                    rs.clear();
                    break;
            }
        }
        lens.add(getLength(ls, rs));
        
        int max = 0;
        for (int i = 0; i < lens.size(); i++) {
            if (lens.get(i) > max) {
                max = lens.get(i);
            }
        }
        
        System.out.print(max);
    }
    
    private static int getLength(Stack<Character> ls, Stack<Character> rs) {
        int len = 0, max = 0;
        while (rs.size() > 0 && ls.size() > 0) {
            char l = ls.pop();
            char r = rs.pop();
            if (getType(l) == getRightType(r)) {
                len += 2;
            }
            else {
                rs.push(r);
                if (len > max) {
                    max = len;
                }
                len = 0;
            }
        }
        return max;
    }
    
    private static int getLastType(Stack<Integer> stack) {
        if (stack.size() > 0) {
            stack.pop();
        }
        if (stack.size() > 0) {
            return stack.peek();
        }
        else {
            return 0;
        }
    }
    
    private static int getType(char c) {
        switch(c) {
            case '(':
                return 1;
            case '[':
                return 2;
            case '{':
                return 3;
            default:
                return 0;
        }
    }
    
    private static int getRightType(char c) {
        switch(c) {
            case ')':
                return 1;
            case ']':
                return 2;
            case '}':
                return 3;
            default:
                return 0;
        }
    }
}
```
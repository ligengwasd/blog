图



# 1、递归解法

思路：

先固定一个字符，然后将固定的字符与它后面的每一个进行交换，一直递归下去，直到固定的字符后面只有一个字符

我们先看看图，框外面的字符是被固定的字符，框里面的字符的没有被固定的字符，具体做法就是每次将框里面的第一个字符与框里面的字符交换（框里面第一个与第一个交换，第一个与第2个交换，第一个与第3个交换.........第1个与第n个交换），直到框里面只剩下一个字符的时候，输出此时的字符排列，但是输出之后又要将字符的位置还原会来。。。（我觉得我讲的有点不太好理解），所以外面现在直接来对图分析吧

假设有abc三个字符，求全排列

看第0层，abc三个字符都在框里面，所以将第一个字符a和第一个字符，第二个字符，第三个字符交换得到：abc，bac，cba，这三个字符串构成了第1层，现在第一层的框里面还有两个字符，所以外面应该继续递归，直到框里面还剩下一个字符就输出这个字符串，所以第一层的abc字符串bc还在框里面，所以将b和b交换，将b和c交换，一共两种情况，（框里面第一个与第一个交换，第一个与第2个交换，第一个与第3个交换.........第1个与第n个交换，一共n种情况）

全排列可以看做固定前i位，对第i+1位之后的再进行全排列，比如固定第一位，后面跟着n-1位的全排列。那么解决n-1位元素的全排列就能解决n位元素的全排列了





```java
public class Q46_Permutations {
    public List<List<Integer>> permute(int[] nums) {
        List<List<Integer>> res = new ArrayList<>();
        fullArray(nums, 0, nums.length-1, res);
        return res;
    }

    private void fullArray(int[] array, int start, int end,List<List<Integer>> res ) {
        if (start == end) {
            List<Integer> list = new ArrayList<>();
            for (int k:array) {
                list.add(k);
            }
            res.add(list);
        } else {
            for (int i = start; i <= end; i++) {
                swap(array, start, i);
                fullArray(array, start + 1, end, res);
                swap(array, start, i);
            }
        }
    }

    public void swap(int s[], int i, int j) {
        int temp = s[i];
        s[i] = s[j];
        s[j] = temp;
    }
}
```

# 2、非递归解法

思路分析
由于非递归的方法是基于对元素大小关系进行比较而实现的，所以这里暂时不考虑存在相同数据的情况。 
在没有相同元素的情况下，任何不同顺序的序列都不可能相同。不同的序列就一定会有大有小。也就是说，我们只要对序列按照一定的大小关系，找到某一个序列的下一个序列。那从最小的一个序列找起，直到找到最大的序列为止，那么就算找到了所有的元素了。 
好了，现在整体思路是清晰了。可是，要怎么找到这里说的下一个序列呢？这个下一个序列有什么性质呢？ 
T[i]下一个序列T[i+1]是在所有序列中比T[i]大，且相邻的序列。关于怎么找到这个元素，我们还是从一个例子来入手吧。 
现在假设序列T[i] = {6, 4, 2, 8, 3, 1}，那么我们可以通过如下两步找到它的下一个序列。 





看完上面的两个步骤，不知道大家有没有理解。如果不理解，那么不理解的点可能就在于替换点和被替点的寻找，以及之后为什么又要进行反转上。我们一个一个地解决问题吧。

替换点和被替换点的寻找。替换点是从整个序列最后一个位置开始，找到一个连续的上升的两个元素。前一个元素的index就是替换点。再从替换点开始，向后搜寻找到一个只比替换点元素大的被替换点。（如果这里你不是很理解，可以结合图形多思考思考。）
替换点后面子序列的反转。在上一步中，可以看到替换之后的子序列（{8, 2, 1}）是一个递减的序列，而替换点又从小元素换成 了大元素，那么与之前序列紧相邻的序列必定是{8, 2, 1}的反序列，即{1, 2, 8}。 
这样，思路已经完全梳理完了，现在就是对其的实现了。只是为了防止给定的序列不是最小的，那就需要对其进行按从小到大进行排序。

```java
package com.ydb.algorithm.leetcode;

import org.junit.Test;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

/**
 * 非递归解法
 *
 * @Author ligeng
 * @Date 19/1/15
 * @Time 下午9:12
 */
public class Q46_Permutations_2 {
    @Test
    public void test() {
        permute(new int[] {1,2,3});
    }

    public List<List<Integer>> permute(int[] nums) {

        core(nums);
        return null;
    }

    private  void core(int[] array) {
        // 先排序
        Arrays.sort(array);
        System.out.println(Arrays.toString(array)); // 最初始的序列
        do {
            nextArray(array);
            System.out.println(Arrays.toString(array));
        } while (!isLast(array));
    }

    private  int[] nextArray(int[] array) {
        int length = array.length;
        // 寻找替换点
        int cursor = 0;
        for (int i = length - 1; i >= 1; i--) {
            // 找到第一个递增的元素对
            if (array[i - 1] < array[i]) {
                cursor = i - 1; // 找到替换点
                break;
            }
        }

        // 寻找在替换点后面的次小元素
        int biggerCursor = cursor + 1;
        for (int i = cursor + 1; i < length; i++) {
            if (array[cursor] < array[i] && array[i] < array[biggerCursor]) {
                biggerCursor = i;
            }
        }

        // 交换
        swap(array, cursor, biggerCursor);

        // 对替换点之后的序列进行反转
        reverse(array, cursor);

        return array;
    }

    private  void reverse(int[] array, int cursor) {
        int end = array.length - 1;
        for (int i = cursor + 1; i <= end; i++, end--) {
            swap(array, i, end);
        }
    }

    private  boolean isLast(int[] array) {
        int length = array.length;
        for (int i = 1; i < length; i++) {
            if (array[i - 1] < array[i]) {
                return false;
            }
        }
        return true;
    }

    public void swap(int s[], int i, int j) {
        int temp = s[i];
        s[i] = s[j];
        s[j] = temp;
    }


//    作者：Q-WHai
//    来源：CSDN
//    原文：https://blog.csdn.net/lemon_tree12138/article/details/50986990
//    版权声明：本文为博主原创文章，转载请附上博文链接！


}
```




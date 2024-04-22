### BubbleSort -- 冒泡排序

```java
/*
 * 冒泡排序基本概念是：
 * 依次比较相邻的两个数，将小数放在前面，大数放在后面。
 * 即在第一趟：首先比较第 1 个和第 2 个数，将小数放前，大数放后。
 * 然后比较第 2 个数和第 3 个数，将小数放前，大数放后，如此继续，
 * 直至比较最后两个数，将小数放前，大数放后。至此第一趟结束，
 * 将最大的数放到了最后。在第二趟：仍从第一对数开始比较
 * （因为可能由于第 2 个数和第 3 个数的交换，使得第 1 个数不再小于第 2 个数），
 * 将小数放前，大数放后，一直比较到倒数第二个数（倒数第一的位置上已经是最大的），
 * 第二趟结束，在倒数第二的位置上得到一个新的最大数（其实在整个数列中是第二大的数）。
 * 如此下去，重复以上过程，直至最终完成排序。
 */
public class BubbleSort {
	public static void sort(int[] data) {
		for (int i = 0; i < data.length - 1; i++) {
			for (int j = 0; j < data.length - 1 - i; j++) {
				if (data[j] > data[j + 1]) {
					int temp = data[j];
					data[j] = data[j + 1];
					data[j + 1] = temp;
				}
			}
		}
	}
}
```

### QuickSort -- 快速排序

```java
/*
 * 快速排序：
 * 一趟快速排序的算法是：
 * 1）设置两个变量 i、j，排序开始的时候：i=0，j=N-1；
 * 2）以第一个数组元素作为关键数据，赋值给 key，即 key=A [0]；
 * 3）从 j 开始向前搜索，即由后开始向前搜索（j=j-1 即 j--），找到第一个小于 key 的值 A [j]，A [i] 与 A [j] 交换；
 * 4）从 i 开始向后搜索，即由前开始向后搜索（i=i+1 即 i++），找到第一个大于 key 的 A [i]，A [i] 与 A [j] 交换；
 * 5）重复第 3、4、5 步，直到 I=J；
 * (3,4 步是在程序中没找到时候 j=j-1，i=i+1，直至找到为止。
 * 找到并交换的时候 i， j 指针位置不变。另外当 i=j 这过程一定正好是 i+ 或 j- 完成的最后令循环结束。）
 */
public class QuickSort {
    public static void quickSort(int[] arr,int low,int high){
        int i,j,temp,t;
        if(low>high){
            return;
        }
        i=low;
        j=high;
        //temp就是基准位
        temp = arr[low];
 
        while (i<j) {
            //先看右边，依次往左递减
            while (temp<=arr[j]&&i<j) {
                j--;
            }
            //再看左边，依次往右递增
            while (temp>=arr[i]&&i<j) {
                i++;
            }
            //如果满足条件则交换
            if (i<j) {
                t = arr[j];
                arr[j] = arr[i];
                arr[i] = t;
            }
        }
        //最后将基准为与i和j相等位置的数字交换
         arr[low] = arr[i];
         arr[i] = temp;
        //递归调用左半数组
        quickSort(arr, low, j-1);
        //递归调用右半数组
        quickSort(arr, j+1, high);
    }
}
```

### SelectionSort -- 选择排序

```java
/*
 * 选择排序基本思路：
 * 把第一个元素依次和后面的所有元素进行比较。
 * 第一次结束后，就会有最小值出现在最前面。
 * 依次类推
 */
public class SelectionSort {
	public static void sort(int[] data) {
		for (int x = 0; x < data.length - 1; x++) {
			for (int y = x + 1; y < data.length; y++) {
				if (data[y] < data[x]) {
					int temp = data[x];
					data[x] = data[y];
					data[y] = temp;
				}
			}
		}
	}
}
```

### InsertSort -- 插入排序

```java
/*
 * 插入排序基本思想
 * 将 n 个元素的数列分为已有序和无序两个部分，如插入排序过程示例下所示：
 * {{a1}, {a2, a3, a4, ..., an}}
 * {{a1(1), a2(1)}, {a3(1), a4(1), ..., an(1)}}
 * {{a1(n-1）, a2(n-1) , ...}, {an(n-1)}}
 * 每次处理就是将无序数列的第一个元素与有序数列的元素从后往前逐个进行比较，
 * 找出插入位置，将该元素插入到有序数列的合适位置中。
 */
public class InsertSort {
	public static void sort(int[] data) {
		for (int i = 1; i < data.length; i++) {
			for (int j = i; (j > 0) && (data[j] < data[j - 1]); j--) {
				int temp = data[j];
				data[j] = data[j - 1];
				data[j - 1] = temp;
			}
		}
	}
}
```

### MergeSort -- 归并排序

```java
/*
 * 归并操作 (merge)，也叫归并算法，指的是将两个已经排序的序列合并成一个序列的操作。
 * 如设有数列 {6，202，100，301，38，8，1}
 * 初始状态： [6] [202] [100] [301] [38] [8] [1]    比较次数
 *   i=1     [6 202 ] [ 100 301] [ 8 38] [ 1 ]        3
 *   i=2     [ 6 100 202 301 ] [ 1 8 38 ]             4
 *   i=3     [ 1 6 8 38 100 202 301 ]                 4
 */
public class MergeSort {
	public static void sort(int[] data) {
		int[] temp = new int[data.length];
		mergeSort(data, temp, 0, data.length - 1);
	}

	private static void mergeSort(int[] data, int[] temp, int l, int r) {
		int mid = (l + r) / 2;
		if (l == r)
			return;
		mergeSort(data, temp, l, mid);
		mergeSort(data, temp, mid + 1, r);

		for (int i = l; i <= r; i++) {
			temp[i] = data[i];
		}
		int i1 = l;
		int i2 = mid + 1;
		for (int cur = l; cur <= r; cur++) {
			if (i1 == mid + 1)
				data[cur] = temp[i2++];
			else if (i2 > r)
				data[cur] = temp[i1++];
			else if (temp[i1] < temp[i2])
				data[cur] = temp[i1++];
			else
				data[cur] = temp[i2++];
		}
	}
}
```

### HeapSort -- 堆排序

```java
/*
 * 堆是一个顺序存储的完全二叉树。
 * 对于 n 个元素的序列 {R1, R2, …, Rn}。当前元素为 R[i] 的话，
 * 它的左子节点是 R[2i] ；
 * 它的右子节点是 R[2i+1] ；
 * 它的父节点是 R[i/2] 。
 * 其中每个结点的关键字都不大于其孩子结点的关键字，这样的堆称为小根堆。(R[i]≤R[2i] && R[i]≤R[2i+1])
 * 其中每个结点的关键字都不小于其孩子结点的关键字，这样的堆称为大根堆。(R[i]≥R[2i] && R[i]≥R[2i+1])

 * 堆排序利用了大根堆（或小根堆）堆顶记录的关键字最大（或最小）这一特征， 使得在当前无序区中选取最大（或最小）关键字的记录变得简单。
 *
 * 用大根堆排序的基本思想 
 * ⑴ 先将初始文件 R[1..n] 建成一个大根堆，此堆为初始的无序区
 * ⑵ 再将关键字最大的记录 R[1]（即堆顶）和无序区的最后一个记录 R[n] 交换，由此得到新的无序区 R[1..n-1] 和有序区 R[n]，
 * 且满足 R[1..n-1].keys ≤ R[n].key 
 * ⑶ 由于交换后新的根 R[1] 可能违反堆性质，故应将当前无序区 R[1..n-1] 调整为堆。
 * 然后再次将 R[1..n-1] 中关键字最大的记录 R[1] 和该区间的最后一个记录 R[n-1] 交换，由此得到新的无序区 R[1..n-2] 和有序区 R[n-1..n]，
 * 且仍满足关系 R[1..n-2].keys ≤ R[n-1..n].keys，同样要将 R[1..n-2] 调整为堆。直到无序区只有一个元素为止。
 * 
 * 大根堆排序算法的基本操作：
 * ⑴ 初始化操作：将 R[1..n] 构造为初始堆；
 * ⑵ 每一趟排序的基本操作：将当前无序区的堆顶记录 R[1] 和该区间的最后一个记录交换，然后将新的无序区调整为堆（亦称重建堆）。
 */
public class HeapSort {

    public static void sort(int[] data) {
        MaxHeap h = new MaxHeap();
        h.init(data);
        for (int i = 0; i < data.length; i++)
            h.remove();
        System.arraycopy(h.queue, 1, data, 0, data.length);
    }

    private static class MaxHeap {
        private int size = 0;
        private int[] queue;

        // 构造初始大根堆
        void init(int[] data) {
            this.queue = new int[data.length + 1];
            for (int i = 0; i < data.length; i++) {
                queue[++size] = data[i];
                fixUp(size);
            }
        }

        public int get() {
            return queue[1];

        }

        public void remove() {
            MaxHeap.swap(queue, 1, size--);
            fixDown(1);
        }

        // 重构大根堆
        private void fixDown(int k) {
            int j;
            while ((j = k << 1) <= size) {
                if (j < size && queue[j] < queue[j + 1])
                    j++;
                if (queue[k] > queue[j]) // 不用交换
                    break;
                MaxHeap.swap(queue, j, k);
                k = j;
            }
        }

        // 找出最大值
        private void fixUp(int k) {
            while (k > 1) {
                int j = k >> 1;
                if (queue[j] > queue[k])
                    break;
                MaxHeap.swap(queue, j, k);
                k = j;
            }
        }

        public static void swap(int[] data, int i, int j) {
            int temp = data[i];
            data[i] = data[j];
            data[j] = temp;
        }
    }
}
```

### ShellSort -- 希尔排序

```java
/*
 * 希尔排序：先取一个小于 n 的整数 d1 作为第一个增量，
 * 把文件的全部记录分成（n 除以 d1）个组。所有距离为 d1 的倍数的记录放在同一个组中。
 * 先在各组内进行直接插入排序；然后，取第二个增量 d2<d1 重复上述的分组和排序，
 * 直至所取的增量 dt=1 (dt<dt-l<…<d2<d1)，即所有记录放在同一组中进行直接插入排序为止。
 *
 * 希尔排序属于插入类排序，是将整个无序列分割成若干小的子序列分别进行插入排序。
 * 排序过程：先取一个正整数 d1<n，把所有序号相隔 d1 的数组元素放一组，组内进行直接插入排序；
 * 然后取 d2<d1，重复上述分组和排序操作；直至 di=1，即所有记录放进一个组中排序为止。
 * 
 * 初始：d=5，49 38 65 97 76 13 27 49 55 04
 *           49 13 |--------------------|
 *           38 27     |---------------------|
 *           65 49          |--------------------|
 *           97 55               |--------------------|
 *           76 04                    |---------------------|
 * 一趟结果 13 27 49 55 04 49 38 65 97 76
 *      d=3，13 27 49  55 04 49 38 65 97 76
 *           13 55 38 76 |------------|------------|------------|
 *           27 04 65           |------------|-----------|
 *           49 49 97               |------------|------------|
 * 二趟结果 13 04 49 38 27 49 55 65 97 76
 *      d=1，13 04 49 38 27 49 55 65 97 76
 * 三趟结果 04 13 27 38 49 49 55 65 76 97
 */
public class ShellSort {
	public static void sort(int[] data) {
		for (int i = data.length / 2; i > 2; i /= 2) {
			for (int j = 0; j < i; j++) {
				insertSort(data, j, i);
			}
		}
		insertSort(data, 0, 1);
	}

	private static void insertSort(int[] data, int start, int inc) {
		for (int i = start + inc; i < data.length; i += inc) {
			for (int j = i; (j >= inc) && (data[j] < data[j - inc]); j -= inc) {
				int temp = data[j];
				data[j] = data[j - inc];
				data[j - inc] = temp;
			}
		}
	}
}
```


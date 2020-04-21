## java中的排序函数

这两天刚刚学习了八种基本的排序算法，一般来说，在自己设计排序算法时，优先选择快排，但是快排也是需要优化的，否则就会出现糟糕的O（n^2）的情况，并且在实际语言中的排序，都按各种情况采用了不同的排序策略。趁热打铁，那么就来分析一下Arrays.sort()函数采用的是什么策略。



## Arrays.sort分析

这里是以int[]型数组为参数的分析。

### （1）：点开sort方法

```java
public static void sort(int[] a) {
        DualPivotQuicksort.sort(a, 0, a.length - 1, null, 0, 0);
    }
```

DualPivotQuicksort类，从单词猜测，java中可能优先采用快排策略，继续。

```java
 static void sort(int[] a, int left, int right,
                     int[] work, int workBase, int workLen) {
        // Use Quicksort on small arrays
        if (right - left < QUICKSORT_THRESHOLD) { //小于286的情况下

            sort(a, left, right, true);
            return;
        }
        
          /*
          在当前数组长度大于286的情况下，采用的是归并排序

         * Index run[i] is the start of i-th run
         * (ascending or descending sequence).
         */
        int[] run = new int[MAX_RUN_COUNT + 1];
        int count = 0; run[0] = left;

        // Check if the array is nearly sorted
        for (int k = left; k < right; run[count] = k) {
            if (a[k] < a[k + 1]) { // ascending
                while (++k <= right && a[k - 1] <= a[k]);
            } else if (a[k] > a[k + 1]) { // descending
                while (++k <= right && a[k - 1] >= a[k]);
                for (int lo = run[count] - 1, hi = k; ++lo < --hi; ) {
                    int t = a[lo]; a[lo] = a[hi]; a[hi] = t;
                }
            } else { // equal
                for (int m = MAX_RUN_LENGTH; ++k <= right && a[k - 1] == a[k]; ) {
                    if (--m == 0) {
                        sort(a, left, right, true);
                        return;
                    }
                }
            }

            /*
             * 当当前分区的范围不大时，即++count=67时，就采用快排
             */
            if (++count == MAX_RUN_COUNT) {
                sort(a, left, right, true);
                return;
            }
        }

        // Check special cases
        // Implementation note: variable "right" is increased by 1.
        if (run[count] == right++) { // The last run contains one element
            run[++count] = right;
        } else if (count == 1) { // The array is already sorted
            return;
        }

        // Determine alternation base for merge
        byte odd = 0;
        for (int n = 1; (n <<= 1) < count; odd ^= 1);

        // Use or create temporary array b for merging
        int[] b;                 // temp array; alternates with a
        int ao, bo;              // array offsets from 'left'
        int blen = right - left; // space needed for b
        if (work == null || workLen < blen || workBase + blen > work.length) {
            work = new int[blen];
            workBase = 0;
        }
        if (odd == 0) {
            System.arraycopy(a, left, work, workBase, blen);
            b = a;
            bo = 0;
            a = work;
            ao = workBase - left;
        } else {
            b = work;
            ao = 0;
            bo = workBase - left;
        }

        // Merging
        for (int last; count > 1; count = last) {
            for (int k = (last = 0) + 2; k <= count; k += 2) {
                int hi = run[k], mi = run[k - 1];
                for (int i = run[k - 2], p = i, q = mi; i < hi; ++i) {
                    if (q >= hi || p < mi && a[p + ao] <= a[q + ao]) {
                        b[i + bo] = a[p++ + ao];
                    } else {
                        b[i + bo] = a[q++ + ao];
                    }
                }
                run[++last] = hi;
            }
            if ((count & 1) != 0) {
                for (int i = right, lo = run[count - 1]; --i >= lo;
                    b[i + bo] = a[i + ao]
                );
                run[++last] = right;
            }
            int[] t = a; a = b; b = t;
            int o = ao; ao = bo; bo = o;
        }
    }
```

发现sort函数，那么接下来就分析sort函数。

### (2):sort

```java
 private static void sort(int[] a, int left, int right, boolean leftmost) {
        int length = right - left + 1;
        //INSERTION_SORT_THRESHOLD=47,即数组长度小于47时，就采用插入排序

        // Use insertion sort on tiny arrays
        if (length < INSERTION_SORT_THRESHOLD) {
            if (leftmost) { //如果从最左开始，就使用插入排序

                /*
                 * Traditional (without sentinel) insertion sort,
                 * optimized for server VM, is used in case of
                 * the leftmost part.
                 */
                for (int i = left, j = i; i < right; j = ++i) {
                    int ai = a[i + 1];
                    while (ai < a[j]) {  //在有序区插入当前位置元素的过程

                        a[j + 1] = a[j];
                        if (j-- == left) {
                            break;
                        }
                    }
                    a[j + 1] = ai;
                }
            } else {
                /*
                 * Skip the longest ascending sequence.
                 */
                do {
                    if (left >= right) {
                        return;
                    }
                } while (a[++left] >= a[left - 1]);

                /*
                  相邻部分的每个元素都起着作用

                  的  哨兵   ，因此这可以避免

                   每次迭代的左范围检查。 而且，我们使用
                   更优化的算法，所谓的配对插入
                   排序，速度更快（在Quicksort的上下文中）
                  比传统的插入排序实现。
                 */
                for (int k = left; ++left <= right; k = ++left) {
                    int a1 = a[k], a2 = a[left];

                    if (a1 < a2) {
                        a2 = a1; a1 = a[left];
                    }
                    while (a1 < a[--k]) {
                        a[k + 2] = a[k];
                    }
                    a[++k + 1] = a1;

                    while (a2 < a[--k]) {
                        a[k + 1] = a[k];
                    }
                    a[k + 1] = a2;
                }
                int last = a[right];

                while (last < a[--right]) {
                    a[right + 1] = a[right];
                }
                a[right + 1] = last;
            }
            return;
        }

        // Inexpensive approximation of length / 7
        int seventh = (length >> 3) + (length >> 6) + 1;

        /*
         围绕（并包括）五个均匀间隔的元素
  *范围中的中心元素。 这些元素将用于
  *枢轴选择如下所述。 间距的选择
  *这些要素是根据经验确定可以在
  *各种各样的输入。   （即快排分区点采用的是五点分区法）。
         */
        int e3 = (left + right) >>> 1; // The midpoint
        int e2 = e3 - seventh;
        int e1 = e2 - seventh;
        int e4 = e3 + seventh;
        int e5 = e4 + seventh;

        // Sort these elements using insertion sort
        if (a[e2] < a[e1]) { int t = a[e2]; a[e2] = a[e1]; a[e1] = t; }

        if (a[e3] < a[e2]) { int t = a[e3]; a[e3] = a[e2]; a[e2] = t;
            if (t < a[e1]) { a[e2] = a[e1]; a[e1] = t; }
        }
        if (a[e4] < a[e3]) { int t = a[e4]; a[e4] = a[e3]; a[e3] = t;
            if (t < a[e2]) { a[e3] = a[e2]; a[e2] = t;
                if (t < a[e1]) { a[e2] = a[e1]; a[e1] = t; }
            }
        }
        if (a[e5] < a[e4]) { int t = a[e5]; a[e5] = a[e4]; a[e4] = t;
            if (t < a[e3]) { a[e4] = a[e3]; a[e3] = t;
                if (t < a[e2]) { a[e3] = a[e2]; a[e2] = t;
                    if (t < a[e1]) { a[e2] = a[e1]; a[e1] = t; }
                }
            }
        }

        // Pointers
        int less  = left;  // The index of the first element of center part
        int great = right; // The index before the first element of right part

        if (a[e1] != a[e2] && a[e2] != a[e3] && a[e3] != a[e4] && a[e4] != a[e5]) {
            /*
      使用五个排序元素中的第二个和第四个作为枢轴。
  *这些值是第一个和第二个的便宜近似值
  *第二个数组。 请注意，pivot1 <=枢轴2。
  （即采用了双轴快排法，选取五个排序元素中的二、四作为分区点）
             */
            int pivot1 = a[e2];
            int pivot2 = a[e4];

            /*
             * 将要排序的第一个和最后一个元素移到
  *以前由枢轴占据的位置。 分区时
  *完成后，枢轴将交换回其最终位置
  *职位，并从后续排序中排除。
             */
            a[e2] = a[left];
            a[e4] = a[right];

            /*
             * Skip elements, which are less or greater than pivot values.
             */
            while (a[++less] < pivot1);
            while (a[--great] > pivot2);

            /*
             * Partitioning:
             *
             *   left part           center part                   right part
             * +--------------------------------------------------------------+
             * |  < pivot1  |  pivot1 <= && <= pivot2  |    ?    |  > pivot2  |
             * +--------------------------------------------------------------+
             *               ^                          ^       ^
             *               |                          |       |
             *              less                        k     great
             *
             * Invariants:
             *
             *              all in (left, less)   < pivot1
             *    pivot1 <= all in [less, k)     <= pivot2
             *              all in (great, right) > pivot2
             *
             * Pointer k is the first index of ?-part.
             */
            outer:
            for (int k = less - 1; ++k <= great; ) {
                int ak = a[k];
                if (ak < pivot1) { // Move a[k] to left part
                    a[k] = a[less];
                    /*
                     * Here and below we use "a[i] = b; i++;" instead
                     * of "a[i++] = b;" due to performance issue.
                     */
                    a[less] = ak;
                    ++less;
                } else if (ak > pivot2) { // Move a[k] to right part
                    while (a[great] > pivot2) {
                        if (great-- == k) {
                            break outer;
                        }
                    }
                    if (a[great] < pivot1) { // a[great] <= pivot2
                        a[k] = a[less];
                        a[less] = a[great];
                        ++less;
                    } else { // pivot1 <= a[great] <= pivot2
                        a[k] = a[great];
                    }
                    /*
                     * Here and below we use "a[i] = b; i--;" instead
                     * of "a[i--] = b;" due to performance issue.
                     */
                    a[great] = ak;
                    --great;
                }
            }

            // Swap pivots into their final positions
            a[left]  = a[less  - 1]; a[less  - 1] = pivot1;
            a[right] = a[great + 1]; a[great + 1] = pivot2;

            // Sort left and right parts recursively, excluding known pivots
            sort(a, left, less - 2, leftmost);
            sort(a, great + 2, right, false);

          ...............
        }
    }
```

### 总结：

在1.8版本的排序中，当元素小于47的情况下，就选择插入排序，当元素大于47小于286的情况下，就优先使用快排，当然快排递归到分区规模小于47时，仍采用的是选择排序。  并且这里的快排是双轴快排法，即采用了五点分区法，每次比较取出二，四两个位置上的元素作为分区点pivot进行分区。而且在插入排序中，也使用了哨兵节点记录上一次的位置，进行了排序优化。

当数据量大于286时，就采用归并排序，当然，在数据量很小的时候，也是采用了插入排序。



### Collections.sort()

其实集合工具类的排序就封装了Arrays.sort(),下面是默认的核心方法

```java
default void sort(Comparator<? super E> c) {
        Object[] a = this.toArray();
        Arrays.sort(a, (Comparator) c);
        ListIterator<E> i = this.listIterator();
        for (Object e : a) {
            i.next();
            i.set((E) e);
        }
    }
    //Arrays.sort
public static <T> void sort(T[] a, Comparator<? super T> c) {
        if (c == null) { //如果是基本数据类型，就默认排序

            sort(a);
        } else {
            if (LegacyMergeSort.userRequested)
                legacyMergeSort(a, c); //快排

            else
                TimSort.sort(a, 0, a.length, c, null, 0, 0);//归并

        }
    }
```

就是先将集合转为对象数组，然后按是否有比较规则进行排序，如果没有compare比较规则，即存储的是基本的数据类型，那么就使用默认的sort（）规则。 

如果是对象的数组类型，那么还是同理，

- 在数据量小于某个值时，使用快排，当然分区至较小时，使用插入排序。

- 在数据量大于某个值时，使用归并，TimSort一种归并排序，同样，归并到较小时，使用插入排序。















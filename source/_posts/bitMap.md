## BitMap

## 数据结构
  使用一个bit位表示一个状态,对于只关注两个状态的需求,使用bitMap更加节省空间.  
比如需要判断一亿数字是否重复,但使用HashSet内存装不下时.bloomFilter也用来判重,  
原理有点类似,如果同时多使用几个bit位来表达,可以更加节省空间.但会有哈希冲突导致误判  

## bitMap使用
  代码片段如下  
~~~java
public class BitMap {

    private static final int SHIFT = 5; //因为2^5 = 32
    private static final int MASK = 8 - 1;

    private static final int[] bitMap = new int[2];
    public static void main(String[] args) {
        set(45);
        System.out.println(test(45));
        clean(45);
        System.out.println(test(45));
        System.out.println(test(46));
        set(46);
        System.out.println(test(46));
        //<< 比 & 优先级高,要用括号
//        System.out.println(1<<(45&7));
//        System.out.println(1<<45&7);

//        System.out.println(~1<<(45&7));
//        System.out.println(Integer.toBinaryString(~(1<<(45&7))));
//        System.out.println(Integer.toBinaryString((1<<(45&7))));
    }

    private static void set(int x){
        //1.确定第几个int,使用x/32 使用位运算就是 x>>5
        //2.确定几个bit, 使用x%8, 使用位运算就是 x&(8-1)
        //3.设置对应的bit为1, 对1进行位移运算
        bitMap[x>>5] |= 1<<(x&7);
    }

    private static boolean test(int x){
        return (bitMap[x>>5] & (1<<(x&7))) != 0;
    }

    private static void clean(int x){

        bitMap[x>>5] &= ~(1<<(x&7));

    }

}
~~~
  比较麻烦的是位预算,与位运算优先级,需要多调试下才可以正确输出

## 补充知识

### 求余数
    对于2^n次方的数取模,有一种快速计算方法 x&(2^n - 1)  
首先需要理解一些知识点:  
1. x/2可以使用 x>>1,把最后一位移除.就如同除以10就是把个位数去掉一样,在二进制中就是把最后一位移除  
221=2x100+2x10+1 , 221/10 = 2x10 + 2x1 + 0, 所以移除最后一位比较好理解了,同理二进制也一样  
2. 对于x/2^n 等价于 x>>n, 所以对于x来说,从第n bit到最左边为x/2^n的商, 而低位的n bit就是余数  
,所以使用位与将低n bit的值取出来 

## 其它
1. 判断x是否为2^n , x&(x-1) == 0
2. 求比x大的2^n次方,java HashMap的方法
~~~java
/**
     * Returns a power of two size for the given target capacity.
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
~~~
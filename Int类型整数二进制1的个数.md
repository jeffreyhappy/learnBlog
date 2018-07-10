```
  /**
   * 给定一个int，返回int转换为2进制后，包含1的个数
   * @param in
   * @return
   */
  public static int findOneNumber(int in){
      int i = 1;
      int count = 0;
      while ( i <= in){
          int result = in & i;
          if (result ==  i){
              count++;
          }
          i =  i << 1;
      }
      return count;
  }
```

```
输入 16   二进制 10000      1的个数 1
输入 1000 二进制 1111101000 1的个数 6
```

1. i << 1     一次就是x2 ,所以循环是 2 4 8 ..... 2的n次方
2. 位数多的就是大数 , 1111代表的十进制大于111，小于11111
3. in的二进制位数是x的话  2的(x-1)次方 <= in <= 2的x次方 -1 .
   例如 x的位数是3,in就是100 (十进制4)、101 (十进制5)、111(十进制6)中的一个， 4 <in <7
4. while ( i <= in) 说明in的位数是大于等于i目前所表示的位数的
5. int result = in & i;
   in是10，i是2的话   1010 & 0010 位与后0010 = 2 与i相等
   in是10, i进化到4   1010 & 0100 位于后0000 = 0 与i不等
   这样就筛选出了1的值   

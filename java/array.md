### 含义

数组

### 使用方法

- 声明
  - int [] ids;
  - ids = new int[]{1,2,3,4};
  - String[] names = new string[5];

- 初始化
  - int 型 数组默认值 为 0 
  - 浮点型 为 0.0


### 注意

- 

```
public class Main {
    public  static void main(String[] agrs){

        int [] ids ; //声明
        ids = new int[]{1,2,3,4};
        String[] names = new String[5]; //长度为 5

        //获取数组长度
        int lem = names.length;

        //数组的默认初始化 ,值为 0
        /*
         *  char 类型 默认值： ASCII 码 的 0
         *  bool 类型是 默认值：false
         *  string 类型 默认值 ：null
         */
        int [] arr = new int[4];
        for (int i=0 ; i < arr.length ; i++){
            System.out.println(arr[i]);     // 0 0 0 0
        }

        //数组的内存解析
        /*
            new 则在堆中 heap 创建一个内存地址，开辟内存空间
            声明变量，在栈 stack 中新增一个变量名，
            赋值变量
            在栈 stack 中变量指向内存地址

            垃圾回收
            引用计数算法
            当程序执行完成，变量出栈销毁，那么堆中的内存地址无人引用，则堆中的内存数据也会删除
         */
    }
}

```


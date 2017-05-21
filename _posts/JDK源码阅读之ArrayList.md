# JDK源码阅读之ArrayList

### 重要方法：

- add()
- addAll()
- fastRemove()
- remove()

### 核心方法:

- System.arraycopy(Object src,int srcPos,Object dest,int destPos,int length)
- Array.copyOf(T[] original,int newLength)

在ArrayList中涉及到有关下标的操作时,如`add(int, E)`,`remove(int)`,其实现原理都是`System.arraycopy()`，该方法实现了源数组对象到目的数组对象的复制

```java

public void add(int index,E e){
  rangeCheckForAdd(index);
  ensureCapacityInternal(size+1);
  System.arraycopy(elementData,index,elementData,index+1,size-index);
  elementData[index] = e;
  size++;
}
```

![]([MarkdownImages](https://github.com/yungchow/MarkdownImages)/[image](https://github.com/yungchow/MarkdownImages/tree/master/image)/**arraycopy.gif**)

当我们需要在特定位置增加一个元素时，该位置前的所有元素保持不变，该位置及其之后的所有元素在新的数组中向后移动一位`System.arraycopy(data,index,data,index+1,size-index)` 。若加入的元素是数组，那么需要两次copy，第一次将原数组从index处向后移动array.length位，第二次将新增数组从index处copy

```java
public boolean addAll(int index Collection<? extends E> c){
  Object[] a = c.toArray();
  //在dest数组中需要向后移动的长度
  int numNew = a.length;
  //需要copy从index到原数组末尾的长度
  int numMoved = size - index;
  //先将原数组从index处向后移动a.length位
  System.arraycopy(elementData,index,elementData,index+numNew,numMoved);
  //再讲新增数组从index处插入
  System.arraycopy(a,0,elementData,index,numNew);
  size += newNum;
  return newNum != 0;
}
```

其他需要移动index的方法以此类推
# 1 线性表
- 由同一类型的数据元素构成的有序序列的线性结构
- 线性表中的元素个数就是线性表的长度
## 1.1 顺序表
- 底层采取顺序存储实现
- 插入,删除的平均时间复杂度为 $O(n)$
- 查找的时间复杂度为 $O(1)$
- 顺序表是一种随机存取的存储结构
```c
#include <stdio.h>  
#include <malloc.h>  
  
typedef int E;  
  
struct List  
{  
    // 指向底层数组的指针  
    E * array;  
    int capacity;  
    int size;  
};  
  
typedef struct List* ArrayList;  
  
_Bool init_list(ArrayList list)  
{  
    // 局部变量离开作用域会销毁，不能直接 E arr[10];    
    list->array = malloc(sizeof(E) * 10);  
    if (list->array == NULL) return 0;  
    list->capacity = 10;  
    list->size = 0;  
    return 1;  
}  

_Bool insert_list(ArrayList list, E e, int index)  
{  
    if (index < 0 || index > list->size) return 0;  
    if (list->size == list->capacity) {  
        int new_cap = (list->capacity >> 1) + list->capacity;  
        // new_cap 需要乘元素大小  
        E *new_array = realloc(list->array, new_cap * sizeof(E));  
        if (new_array == NULL) return 0;  
        list->array = new_array;  
        list->capacity = new_cap;  
    }  
    for (int i = list->size; i > index; i--) {  
        list->array[i] = list->array[i - 1];  
    }  
    list->array[index] = e;  
    list->size++;  
    return 1;  
}  
  
_Bool delete_list(ArrayList list, int index)  
{  
    if(index < 0 || index > list->size) return 0;  
    for (int i = index; i < list->size - 1; i++)  
        list->array[i] = list->array[i + 1];  
    list->size--;  
    return 1;  
}  
  
int length(ArrayList list)  
{  
    return list->size;  
}  
  
E * get(ArrayList list, int index)  
{  
    if (index < 0 || index > list->size - 1) return NULL;  
    return &list->array[index];  
}  
  
void print_list(ArrayList list)  
{  
    for (int i = 0; i< list->size; i++)  
        printf("%d  ", list->array[i]);  
    printf("\n");  
}  
  
int find_element(ArrayList list, E e)  
{  
    for (int i = 0; i < list->size; i++)  
    {  
        if (list->array[i] == e)  
        {  
            return i;  
        }  
    }  
    return -1;  
}
```
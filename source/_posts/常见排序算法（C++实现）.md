---
title: 常见排序算法（C++实现）
date: 2016-05-24 13:55
categories: 算法
tags: 排序
---

## 一、直接插入排序

``` cpp
#include<iostream>
using namespace std;

/*
直接插入排序
稳定的排序算法，时间复杂度：O(n^2)
*/
void insertSort(int a[], int n){
    int temp;//存储要插入的数
    int i, j;//j:实际上用来记录位置

/*只需要对后面n-1个数排序*/
    for (i = 1; i < n; i++){
        temp = a[i];// 暂时存储要插入的数
        if (a[i] < a[i - 1]){
            for (j = i - 1; j>=0 && temp < a[j]; j--){//遍历前面的数，进行比较，寻找插入的位置
                a[j + 1] = a[j];//前面大的数后移
            }
            a[j + 1] = temp;// j是刚好不满足情况的位置，则 j+1 满足该位置
        }
    }
}

/*打印数组*/
void print(int a[], int n){
    for (int i = 0; i < n; i++){
        cout << a[i]<<" ";
    }
    cout << endl;
}

void main(){
    int a[] = { 3, 2, 43, 12, 90, 23 };
    int n = sizeof(a) / sizeof(a[0]);
    //cout << n << endl;
    cout << "排序前：" << endl;
    print(a, n);

    insertSort(a, n);

    cout << "排序后：" << endl;
    print(a, n);
    system("pause");
}
```

## 二、折半插入排序 
 

``` cpp
#include<iostream>
using namespace std;

/*
折半插入排序：时间复杂度：O(n^2)，稳定
对直接插入排序 “初步优化”，寻找插入位置更快
*/
void binaryInsertSort(int a[], int n) {
    int temp;
    int i, j;
    for (i = 1; i < n; i++) {
        if (a[i] < a[i - 1]) {
            temp = a[i];

            // “二分法”，寻找插入位置，存储到low
            int low = 0, high = i - 1, mid;
            while (low <= high) {
                mid = (low + high) / 2;
                if (temp < a[mid]) {
                    high = mid - 1;
                }
                else {
                    low = mid + 1;
                }
            }

            // 对要插入数之前小的数，整体后移
            for (j = i - 1; j >= low; j--) {
                a[j + 1] = a[j];
            }

            // 最后将刚要插入的数放到找到的位置上
            a[low] = temp;
        }
    }
}

/*打印数组*/
void print(int a[], int n) {
    for (int i = 0; i < n; i++) {
        cout << a[i] << " ";
    }
    cout << endl;
}

void main() {
    int a[] = {3, 2, 488, 12, 90, 23};
    int n = sizeof(a) / sizeof(a[0]);
    cout << "排序前：" << endl;
    print(a, n);

    binaryInsertSort(a, n);

    cout << "排序后：" << endl;
    print(a, n);
    system("pause");
}
```

## 三、希尔排序

``` cpp
#include<iostream>
using namespace std;

/*
希尔排序：不稳定的排序
对直接插入排序的 “终极优化”，分组进行直接插入排序
时间复杂度： O(n^1.3),范围： [O(n^2), O(nlog2(n))]
*/
void shellSort(int a[],int n){
    int d;// 数之间的间隔
    int temp;// 存储要插入的数
    int i, j;
    // 先对要排序的数，进行分组,间隔直到1
    for (d = n / 2; d >= 1; d /= 2){
        // i++,对每个分组交替执行排序，并不是一整个组排完
        for (i = d; i < n; i++){// 也是从该组的第二个数开始，和后面的数比较
            temp = a[i];

            for (j = i - d; j >= 0 && temp < a[j]; j -= d){
                a[j + d] = a[j];
            }
            a[j + d] = temp;
        }
    }
}

/*打印数组*/
void print(int a[], int n){
    for (int i = 0; i < n; i++){
        cout << a[i]<<" ";
    }
    cout << endl;
}

void main(){
    int a[] = { 13, 0, 43, 12, 1, 23 };
    int n = sizeof(a) / sizeof(a[0]);
    //cout << n << endl;
    cout << "排序前：" << endl;
    print(a, n);

    shellSort(a, n);

    cout << "排序后：" << endl;
    print(a, n);
    system("pause");
}
```

## 四、冒泡排序

``` cpp
#include<iostream>
using namespace std;
// 提前声明
void Swap(int &a, int &b);

/*
冒泡排序O(n^2)
稳定的排序
*/
void bubbleSort(int a[], int n){
    // n-1趟
    for (int i = 0; i < n - 1; i++){
        for (int j = i; j < n - 1 - i; j++){// 注意最后两个数
            if (a[j]>a[j + 1]){
                Swap(a[j], a[j + 1]);
            }
        }
    }
}

/*交换两个数*/
void Swap(int &a, int &b){
    a = a^b;
    b = a^b;
    a = a^b;
}

/*打印数组*/
void print(int a[], int n){
    for (int i = 0; i < n; i++){
        cout << a[i] << " ";
    }
    cout << endl;
}

void main(){
    int a[] = { 3, 2, 28, 12, 90, 23 };
    int n = sizeof(a) / sizeof(a[0]);
    cout << "排序前：" << endl;
    print(a, n);

    bubbleSort(a, n);

    cout << "排序后：" << endl;
    print(a, n);
    system("pause");
}
```

## 五、快速排序

``` cpp
#include<iostream>
using namespace std;
int partition(int a[], int p, int r);

/*
快速排序,对冒泡排序的改进,不稳定
O(nlog2n),最差情况下：退化为冒泡排序O(n^2)
*/
void quickSort(int a[], int left,int right){
    if (left < right){
        // 递归进行排序
        int q = partition(a, left, right);
        quickSort(a, left, q - 1);//左半边
        quickSort(a, q + 1, right);//右半边
    }
}

/*选取基准元素key，进行划分*/
int partition(int a[], int p, int r){
    int i = p;
    int j = r;
    int key = a[i];

    while (i<j) {
        while (i < j && key < a[j]){//右侧扫描
            j--;
        }
        a[i] = a[j];
        while (i < j && a[i] < key){//左侧扫描
            i++;
        }
        a[j] = a[i];
    }
    a[i] = key;//一轮循环完，找到key的位置
    return i;
}

/*打印数组*/
void print(int a[], int n){
    for (int i = 0; i < n; i++){
        cout << a[i] << " ";
    }
    cout << endl;
}

void main(){
    int a[] = { 3, 2, 28, 12, 90, 23 };
    int n = sizeof(a) / sizeof(a[0]);
    cout << "排序前：" << endl;
    print(a, n);

    quickSort(a, 0, n-1);

    cout << "排序后：" << endl;
    print(a, n);
    system("pause");
}
```

## 六、选择排序

``` cpp
using namespace std;
void Swap(int &a,int &b);

/*
简单选择排序稳定
时间复杂度：O(n^2)
*/
void selectionSort(int a[], int n){
// n-1趟，每次选出一个最小的放入有序区
    for (int i = 0; i < n - 1; i++){
        for (int j = i+1; j < n; j++){
            if (a[i]>a[j]){
                Swap(a[i], a[j]);
            }
        }
    }
}

/*交换两个数*/
void Swap(int &a, int &b){
    a = a^b;
    b = a^b;
    a = a^b;
}

/*打印数组*/
void print(int a[], int n){
    for (int i = 0; i < n; i++){
        cout << a[i] << " ";
    }
    cout << endl;
}

void main(){
    int a[] = { 3, 2, 28, 12, 90, 23 };
    int n = sizeof(a) / sizeof(a[0]);
    cout << "排序前：" << endl;
    print(a, n);

    selectionSort(a, n);

    cout << "排序后：" << endl;
    print(a, n);
    system("pause");
}
```

## 七、堆排序

``` cpp
#include<iostream>
using namespace std;

/*
    堆排序：        所有情况下：O(nlog2n)
    不稳定，        对选择排序的一种改进
    （下面采用大根堆方法）
*/

/*交换两个数*/
void Swap(int &a, int &b){
    a = a^b;
    b = a^b;
    a = a^b;
}

/*堆的筛选算法*/
void sift(int a[], int start, int end){
    int i = start;
    for (int j = 2 * i + 1; j <= end; j = 2 * j + 1){// 重复操作至叶节点
        if (j < end && a[j] < a[j + 1]){// 选出孩子节点最大的，后面来比较
            j++;
        }
        if (a[i] >= a[j]){// 已经是堆了，不用调整
            break;
        }else{
            Swap(a[i], a[j]);// 最大孩子上移至根节点
            i = j;
        }
    }
}

/*堆排序*/
void heapSort(int a[], int n){
    for (int i = n / 2; i >= 0; i--){// 初始建堆，完全二叉树，只需从  n/2 中间节点处开始，向上
        sift(a, i, n-1);
    }
    for (int i = n - 1; i > 0;i--){// 根节点和每次最后一个节点交换
        Swap(a[0], a[i]);
        sift(a, 0, i-1);
    }
}

/*打印数组*/
void print(int a[], int n){
    for (int i = 0; i < n; i++){
        cout << a[i] << " ";
    }
    cout << endl;
}

void main(){
    int a[] = { 3, 112, 28, 12, 90, 23 };
    int n = sizeof(a) / sizeof(a[0]);
    cout << "排序前：" << endl;
    print(a, n);

    heapSort(a, n);

    cout << "排序后：" << endl;
    print(a, n);
    system("pause");
}
```

## 八、归并排序

``` cpp
#include<iostream>
using namespace std;

/*
归并排序任何情况下： O(nlog2n)空间复杂度O(n)
稳定的排序算法（外排采用）
*/

/*一次归并算法*/
void merge(int a[], int start, int mid, int end){
    int i = start;
    int j = mid + 1;
    int k = 0;
    int *temp = new int[end - start + 1];//临时存放排好序的数组

    while (i <= mid && j <= end){
        if (a[i] < a[j]){
            temp[k++] = a[i++];
        }
        else{
            temp[k++] = a[j++];
        }
    }

    //处理其中一个没有取完的序列
    if (i <= mid){
        while (i <= mid){
            temp[k++] = a[i++];
        }
    }
    else{
        while (j <= end){
            temp[k++] = a[j++];
        }
    }

    // 将临时数组有序数据赋值给原数组
    for (i = start,k=0; i <= end; i++,k++){
        a[i] = temp[k];
    }

    // 释放内存空间
    delete []temp;//此处是数组，必须是[]
}

/*归并排序递归*/
void mergeSort(int arr[], int low, int high)
{
    if (low<high)
    {
        int mid = (low + high) / 2;
        mergeSort(arr, low, mid);
        mergeSort(arr, mid + 1, high);
        // 将最后两个有序的序列合并
        merge(arr, low, mid, high);
    }
}

/*打印数组*/
void print(int a[], int n){
    for (int i = 0; i < n; i++){
        cout << a[i] << " ";
    }
    cout << endl;
}

void main(){
    int a[] = { 3, 112, 0, 12, 90, 23 };
    int n = sizeof(a) / sizeof(a[0]);
    cout << "排序前：" << endl;
    print(a, n);

    mergeSort(a, 0, n - 1);

    cout << "排序后：" << endl;
    print(a, n);
    system("pause");
}
```


---
title: shell命令使用数组循环
---

shell 数组循环的用法，对于一些构建工具很有帮助

``` bash
 #！/bin/bash

#定义方法一  数组定义为空格分割
array=('a' 'b' 'c' 'd' 'e')

#定义方法二  
arrayIndex[0]=1
arrayIndex[1]=2
arrayIndex[2]=3
arrayIndex[3]=4
arrayIndex[4]=5

#修改数组值
array[0]='f'
arrayIndex[1]=6

#打印数组长度
echo ${#arrayIndex[@]}

#for 遍历数组
for var in ${arrayIndex[@]}
do
  echo $var
done

#while 遍历数组
i=0
while  [[ i -lt ${#arrayIndex[@]} ]];do
  echo ${arrayIndex[i]}
  let i++
done
```
## 使用shell做服务器的健康检查脚本

``` bash
# health.sh

# 这个地方配置多个要进行检查的URL，一般的，这些URL都是来自于某个系统的
check_urls=(
  'https://wiki.enbrands.com/display/IMX/hello' 
  'https://wiki.enbrands.com/display/IMX/hello'
)

for url in ${check_urls[@]}
do
  curl url
  # 进行请求，如果请求失败则直接将程序标记为失败
done

```
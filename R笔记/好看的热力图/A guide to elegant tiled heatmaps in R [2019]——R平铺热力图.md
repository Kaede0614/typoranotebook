# 【小刻也能看懂的R数据可视化】优雅的平铺热力图指南（翻译）

> 原文链接：https://www.royfrancis.com/a-guide-to-elegant-tiled-heatmaps-in-r-2019/
>
> 包含数据集来源与下载。

- 数据集：~~源石病~~ 麻疹一级发病率（每10万人）
- 目的：绘制简洁、干净、优雅的热力图
- 用到的包：==ggplot2==、==dplyr==​、==tidyr==、==stringr​==

```R
# install packages
# install.packages(pkgs = c("ggplot2","dplyr","tidyr","stringr","gplots","plotrix"),dependencies = T)

# load packages
library(ggplot2) # ggplot() for plotting
library(dplyr) # data reformatting
library(tidyr) # data reformatting
library(stringr) # string manipulation
```

## 1.数据准备

首先导入csv文件并检查数据，跳过csv表格的前两行。

```R
# read csv file
m <- read.csv("measles_lev1.csv",header=T,stringsAsFactors=F,skip=2)

# inspect data
head(m)
str(m)
table(m$YEAR)
table(m$WEEK)
```

- ==head()==查看前6行数据。
- ==str()==检查数据结构。发现年和月是int，发生率是chr。
- ==table()==检查年和周是否有缺失数据。

目前数据是“宽”格式，而==ggplot2==需要“长”格式，因此需要转换。年和月保持原样，所有发生率折叠到一列。为了方便将列名改为小写。年和月变量转换为factor，值转换为数字。

```R
m2 <- m %>%
  # convert data to long format
  gather(key="state",value="value",-YEAR,-WEEK) %>%
  # rename columns
  setNames(c("year","week","state","value")) %>%
  # convert year to factor
  mutate(year=factor(year)) %>%
  # convert week to factor
  mutate(week=factor(week)) %>%
  # convert value to numeric (also converts '-' to NA, gives a warning)
  mutate(value=as.numeric(value))
```

这样处理后数据集变成适合==ggplot2==处理的格式。

```R
> head(m2)
  year week   state value
1 1928    1 Alabama  3.67
2 1928    2 Alabama  6.25
3 1928    3 Alabama  7.95
4 1928    4 Alabama 12.58
5 1928    5 Alabama  8.03
6 1928    6 Alabama  7.27
```

该m2数据集如果导出为csv，格式如下。

<table>
   <tr>
      <td></td>
      <td>        year</td>
      <td>       week</td>
      <td>             state  </td>
      <td>                     value</td>
   </tr>
   <tr>
      <td>33325</td>
      <td>1960</td>
      <td>45</td>
      <td>DISTRICT.OF.COLUMBIA</td>
      <td>NA</td>
   </tr>
   <tr>
      <td>33326</td>
      <td>1960</td>
      <td>46</td>
      <td>DISTRICT.OF.COLUMBIA</td>
      <td>0.13</td>
   </tr>
   <tr>
      <td>33327</td>
      <td>1960</td>
      <td>47</td>
      <td>DISTRICT.OF.COLUMBIA</td>
      <td>0.13</td>
   </tr>
   <tr>
      <td>33328</td>
      <td>1960</td>
      <td>48</td>
      <td>DISTRICT.OF.COLUMBIA</td>
      <td>0.26</td>
   </tr>
   <tr>
      <td>33329</td>
      <td>1960</td>
      <td>49</td>
      <td>DISTRICT.OF.COLUMBIA</td>
      <td>NA</td>
   </tr>
   <tr>
      <td>33330</td>
      <td>1960</td>
      <td>50</td>
      <td>DISTRICT.OF.COLUMBIA</td>
      <td>2.09</td>
   </tr>
   <tr>
</table>
可以看出各州名称是全大写，并且单词间用点分隔。美观考虑修改为首字母大写（Title Case），空格分隔。考虑用==str_to_title()==处理单词再将他们粘贴到一起。

```R
# removes . and change states to title case using custom function
fn_tc <- function(x) paste(str_to_title(unlist(strsplit(x,"[.]"))),collapse=" ")
m2$state <- sapply(m2$state,fn_tc)
```

- ==strsplit()==：按分隔符拆分字符串。注意由于“.”在R中有特殊含义，需要用正则表达式表示（[链接](https://stackoverflow.com/questions/26665100/how-to-use-the-strsplit-function-with-a-period)）。
- ==unlist()==：将列表转换为向量，以便运算。
- ==str_to_title==：将英文字符串中的单词首字母大写。
- ==paste==：将字符串连接起来，分隔符为空格。注意collpase负责的是字符串内部的连接，两组字符串连接用sep。
- R中匿名函数最常用于==*apply()==类函数。
- ==sapply()==：返回处理后的向量，替代m2的州名。

接下来是绘图。考虑X轴为年，Y轴为各州名称，这意味着要先处理周这个变量。我们将每年所有星期的发病率相加，再删除周变量。==dplyr==的方法是用函数==group_by()==和==summarise()==。

==sum()==函数处理==NA==的方式比较奇怪，默认下，如果输入向量中有==NA==就会返回==NA==。如果设置参数==na.rm=TRUE==，那么==NA==会被移除并加总剩余数字。但是如果所有元素均为==NA==，那么会返回0。这不适合本例，因此选择自定义加总函数==na_sum()==来移除==NA==，并且如果所有元素为==NA==时返回==NA==。接着在==summarise()==函数中使用该自定义函数，按年份和州汇总数去，同时去掉周。

```R
# custom sum function returns NA when all values in set are NA,
# in a set mixed with NAs, NAs are removed and remaining summed.
na_sum <- function(x)
{
  if(all(is.na(x))) val <- sum(x,na.rm=F)
  if(!all(is.na(x))) val <- sum(x,na.rm=T)
  return(val)
}

# sum incidences for all weeks into one year
m3 <- m2 %>%
  group_by(year,state) %>%
  summarise(count=na_sum(value)) %>%
  as.data.frame()
```

- ==na.rm==参数为T时移除所有==NA==再加总，为F时返回==NA==。
- ==dplyr==分两步完成汇总。先==group_by()==定义分组变量，再==summarise()==描述如何汇总。

处理后m3中没有周变量，如下。

```R
> head(m3)

  year      state  count
1 1928    Alabama 334.99
2 1928     Alaska   0.00
3 1928    Arizona 200.75
4 1928   Arkansas 481.77
5 1928 California  69.22
6 1928   Colorado 206.98
```

现在数据准备工作基本结束。数据为“长数据”格式，三变量分别为factor、factor、numeric型。



## 2.绘图

### 2.1 ggplot2

先用基础ggplot画一下。

```R
#basic ggplot
p <- ggplot(m3,aes(x=year,y=state,fill=count))+
      geom_tile()

#save plot to working directory
ggsave(p,filename="measles-basic.png")
```

![img](https://www.royfrancis.com/a-guide-to-elegant-tiled-heatmaps-in-r-2019/measles-basic.png)


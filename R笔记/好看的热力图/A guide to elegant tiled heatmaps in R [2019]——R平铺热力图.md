# A guide to elegant tiled heatmaps in R [2019]——R平铺热力图

> 原文链接：https://www.royfrancis.com/a-guide-to-elegant-tiled-heatmaps-in-r-2019/
>
> 包含数据集来源与下载。

- 数据集：麻疹一级发病率（每10万人）
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

## 1. 数据准备

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

接下来是绘图。考虑X轴为年，Y轴为各州名称，这意味着要先处理周这个变量。我们将每年所有星期的发病率相加，再删除周变量。==dplyr==的方法是用函数==group_by()==和==summarise()==。




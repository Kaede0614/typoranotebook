# A guide to elegant tiled heatmaps in R [2019]——R平铺热力图

> 原文链接：https://www.royfrancis.com/a-guide-to-elegant-tiled-heatmaps-in-r-2019/
>
> 包含数据集来源与下载。

- 数据集：麻疹一级发病率（每10万人）
- 目的：简洁、干净、优雅的热力图
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






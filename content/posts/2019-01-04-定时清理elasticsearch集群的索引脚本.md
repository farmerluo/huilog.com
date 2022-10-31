---
title: 定时清理elasticsearch集群的索引脚本
author: 阿辉
date: 2019-01-04T03:25:13+00:00
categories:
- ElasticSearch
tags:
- ElasticSearch
keywords:
- ElasticSearch
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
elasticsearch集群容量总是有限的，所以必需要对超过一定时间的索引进行删除和清理。
先说明下我们索引的命令方式：xxx-xxx-xxx-yyyy.mm.dd
yyyy.mm.dd为日期。

清理脚本如下：
```bash
#!/bin/bash
###################################
#删除早于天的ES集群的索引
###################################
# crontab -e
#clean es index
#* 0 * * * sh /data/shell/clean_es_indes.sh 

#索引保存天数
days=30
 
#ES cluster url
es_cluster_url="http://127.0.0.1:9200"

function delete_indices() {
    comp_date=`date -d "$days day ago" +"%Y-%m-%d"`
    date1="$1 00:00:00"
    date2="$comp_date 00:00:00"

    t1=`date -d "$date1" +%s` 
    t2=`date -d "$date2" +%s` 

    if [ $t1 -le $t2 ]; then
        echo "$1时间早于$comp_date，进行索引删除"
        #转换一下格式，将类似2017-10-01格式转化为2017.10.01
        format_date=`echo $1| sed 's/-/\./g'`
        echo "curl -XDELETE $es_cluster_url/*$format_date"
        curl -s -XDELETE "$es_cluster_url/*$format_date"
    fi
}

curl -s -XGET "$es_cluster_url/_cat/indices" | awk -F" " '{print $3}' | awk -F"-" '{print $NF}' | egrep "[0-9]*\.[0-9]*\.[0-9]*" | sort | uniq  | sed 's/\./-/g' | while read LINE
do
    #调用索引删除函数
    delete_indices $LINE
done
```

<!--more-->

另一个脚本，适用于日期时间格式为xxxx-2019-10-08的索引：

```bash
#!/bin/bash
searchIndex=console-log
elastic_url=10.16.16.36
elastic_port=9200
save_days=7

date2stamp () {
    date --utc --date "$1" +%s
}

dateDiff (){
    case $1 in
        -s)   sec=1;      shift;;
        -m)   sec=60;     shift;;
        -h)   sec=3600;   shift;;
        -d)   sec=86400;  shift;;
        *)    sec=86400;;
    esac
    dte1=$(date2stamp $1)
    dte2=$(date2stamp $2)
    diffSec=$((dte2-dte1))
    if ((diffSec < 0)); then abs=-1; else abs=1; fi
    echo $((diffSec/sec*abs))
}

for index in $(curl -s "${elastic_url}:${elastic_port}/_cat/indices?v" | grep -E " ${searchIndex}-20[0-9][0-9]-[0-1][0-9]-[0-3][0-9]" | awk '{ print $3 }'); do
  date=$(echo ${index: -10} | sed 's/\./-/g')
  cond=$(date +%Y-%m-%d)
  diff=$(dateDiff -d $date $cond)
  #echo -n "${index} (${diff})"
  if [ $diff -gt $save_days ]; then
    echo "curl -XDELETE \"${elastic_url}:${elastic_port}/${index}?pretty\""
    curl -XDELETE "${elastic_url}:${elastic_port}/${index}?pretty"
  else
    echo "skip delete index: ${index}"
  fi
done
```


参考：
https://blog.csdn.net/felix_yujing/article/details/78207667

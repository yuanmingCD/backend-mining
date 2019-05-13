# CPU过高问题排查







通过top命令获取占用CPU资源的进程

```
top
```





查看占用CPU高的线程

```
 ps -mp 41163 -o THREAD,tid,time | sort -nr -t ' ' -k 2
```



获取线程16进制编号

```
printf '%x\n' 41449
```

重定向stack信息到文件中

```
jstack pid > thread.txt
```



通过得到的16进制线程id在文件中搜索，排名前几的线程资源如下




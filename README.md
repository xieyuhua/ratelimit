 
在网站的运营中，经常会遇到需要对用户访问次数做限制的情况，比如非常典型的是对于某些付费访问服务，需要对访问频率做比较精确的限制，比如单个用户(或者每个IP)地址每天只允许访问多少次，然后每小时只允许访问多少次等等，ratelimit就是针对这种情况而设计。
    
不同于网关级限流(包括go.uber.org/ratelimit漏桶限流以及github.com/juju/ratelimit令牌桶限流),本限流方案为业务级限流，适用于平台运营中,精细化的按单个用户,按IP等限流,为业内rdeis滑动窗口限流方案的纯GO替代方案,并且支持持久化(可选),可定期把历史数据备份到本地磁盘,程序重启也可保留之前的访问记录，另外，根据网站实际的运营需求，本库还提供一些比较特殊的函数，诸如ManualEmptyVisitorRecordsOf,允许清空某访客的访问记录，从而达到临时单独增加某用户的访问次数的目的。
      
github.com/yudeguang/ratelimit底层用一个大小能自动伸缩的环形队列来存储用户访问数据，并发安全，拥有较高性能的同时还非常省内存,同时拥有高达1000W次/秒的处理能力(redis约10W次/秒)。作为对比，与用redis的相关数据结构来实现用户访问控制相比，其用法相对简单。 
  
  
##每10秒只允许访问5次,每30分钟只允许访问50次,每天只允许访问500次
```
r.AddRule(time.Second*10, 5)  //每10秒只允许访问5次
r.AddRule(time.Minute*30, 50) //每30分钟只允许访问50次
r.AddRule(time.Hour*24, 500)  //每天只允许访问500次
```

##使用案例如下
```go
package main

import (
	"fmt"
	"log"
	"strconv"
	"sync"
	"time"
	"encoding/json"
    "net/http"
	"github.com/yudeguang/ratelimit"
)

var ratelimits map[string]*ratelimit.Rule

type JsonRes struct {
	Code int         `json:"code"`
	Msg  string      `json:"msg"`
}

func main() {
	log.SetFlags(log.Lshortfile | log.Ltime)
	
    ratelimits = make(map[string]*ratelimit.Rule, 0)
    /*
    *1、程序写死
    *2、文件配置
    *3、读取redis
    */
    
    //ip
    ratelimits["ip"] = ratelimit.NewRule()//步骤一：初始化
    ratelimits["ip"].AddRule(time.Second*10, 100)//每10秒只允许访问100次   规则
    ratelimits["ip"].LoadingAndAutoSaveToDisc("ip", time.Second*10) //从本地磁盘加载历史访问数据
    
    //url
    ratelimits["url"] = ratelimit.NewRule()
    ratelimits["url"].AddRule(time.Minute*30, 1000)//每30分钟只允许访问1000次
    ratelimits["url"].AddRule(time.Hour*24, 5000) //每天只允许访问5000次
    ratelimits["url"].LoadingAndAutoSaveToDisc("url", time.Second*10) 
    

	http.HandleFunc("/reset", reset)
	http.HandleFunc("/", example)
	
	
	log.Println("监听端口", "http://0.0.0.0:8086")
	listenErr := http.ListenAndServe(":8086", nil)
	if listenErr != nil {
		log.Fatal("ListenAndServe: ", listenErr)
	}
}
// 重置
func reset(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("content-type", "text/json")
	defer func() {
		//捕获 panic
		if err := recover(); err != nil {
    		msg, _ := json.Marshal(&JsonRes{Code: 4000, Msg: " 500 reset NOT FOUND !"})
    		w.Write(msg)
		}
	}()
	r.ParseForm() // 解析参数
	qw := r.PostFormValue("qw")
	
	//判断是否有值
	if _, ok := ratelimits[qw]; !ok {
		msg, _ := json.Marshal(&JsonRes{Code: 404, Msg: " 404 qw NOT FOUND !"})
		w.Write(msg)
		return
	}
	
	//chery清空访问记录
    ratelimits[qw].ManualEmptyVisitorRecordsOf(qw)
    
	data := fmt.Sprintf(qw + "清空访问记录后,剩余:%d", ratelimits[qw].RemainingVisits(qw))
	msg, _ := json.Marshal(&JsonRes{Code: 200, Msg: data })
	w.Write(msg)
	return
}


// 访问次数
func example(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("content-type", "text/json")
	defer func() {
		//捕获 panic
		if err := recover(); err != nil {
		    log.Println(err)
    		msg, _ := json.Marshal(&JsonRes{Code: 4000, Msg: " 500 index NOT FOUND !"})
    		w.Write(msg)
		}
	}()
	
	r.ParseForm() // 解析参数
	qw := r.PostFormValue("qw")
	
	//判断是否有值
	if _, ok := ratelimits[qw]; !ok {
		msg, _ := json.Marshal(&JsonRes{Code: 404, Msg: " 404 qw NOT FOUND !"})
		w.Write(msg)
		return
	}
	
    //步骤四：调用函数判断某用户是否允许访问
	allow := ratelimits[qw].AllowVisit(qw)
	log.Println(allow)

	
	//步骤五(可选):程序退出前主动手动存盘
	err := ratelimits[qw].SaveToDiscOnce() //在自动备份的同时，还支持手动备份，一般在程序要退出时调用此函数
	if err == nil {
		log.Println("完成手动数据备份")
	} else {
		log.Println(err)
	}
	
	
	data := fmt.Sprintf(qw+"剩余:%d", ratelimits[qw].RemainingVisits(qw))
	msg, _ := json.Marshal(&JsonRes{Code: 200, Msg: data })
	w.Write(msg)
	return

}
```


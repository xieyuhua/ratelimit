 
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
var  r *ratelimit.Rule

type JsonRes struct {
	Code int         `json:"code"`
	Msg  string      `json:"msg"`
}

func main() {
	log.SetFlags(log.Lshortfile | log.Ltime)
    r = ratelimit.NewRule()
    r.AddRule(time.Second*10, 100)
	r.LoadingAndAutoSaveToDisc("test1", time.Second*10) 
	
	http.HandleFunc("/reset", reset)
	http.HandleFunc("/", index)
	
	log.Println("监听端口", "http://0.0.0.0:8086")
	
	listenErr := http.ListenAndServe(":8086", nil)
	
	if listenErr != nil {
		log.Fatal("ListenAndServe: ", listenErr)
	}
	
}

func reset(w http.ResponseWriter, req *http.Request) {
    w.Header().Set("content-type", "text/json")
	defer func() {
		//捕获 panic
		if err := recover(); err != nil {
			log.Println("查询sql发生错误", err)
    		msg, _ := json.Marshal(&JsonRes{Code: 4000, Msg: " 500 NOT FOUND !"})
    		w.Write(msg)
		}
	}()
	
	
    r.ManualEmptyVisitorRecordsOf("chery")
    
    
    
	data := fmt.Sprintf("chery清空访问记录前,剩余:%d", r.RemainingVisits("chery"))
	msg, _ := json.Marshal(&JsonRes{Code: 200, Msg: data })
	w.Write(msg)
	return
}
func index(w http.ResponseWriter, req *http.Request) {
	w.Header().Set("content-type", "text/json")
	defer func() {
		//捕获 panic
		if err := recover(); err != nil {
			log.Println("查询sql发生错误", err)
    		msg, _ := json.Marshal(&JsonRes{Code: 4000, Msg: " 500 NOT FOUND !"})
    		w.Write(msg)
		}
	}()
	

	allow := r.AllowVisit("chery")
	log.Println(allow)
	
	
	data := fmt.Sprintf("chery清空访问记录前,剩余:%d", r.RemainingVisits("chery"))
	msg, _ := json.Marshal(&JsonRes{Code: 200, Msg: data })
	w.Write(msg)
	return

}
```


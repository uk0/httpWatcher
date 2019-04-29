# httpWatcher
http watch on


```golang
package main

import (
	"fmt"
	"github.com/astaxie/beego/logs"
	"github.com/nahid/gohttp"
	"os"
	"time"
)

type Content struct {
	url     string
	content []byte
}
var MiniBatch = 10;
var outTime int64 = 5;
var start = 0;

type WatchData struct {
	Token string
	Msg   string
	Topic string
}

var (
	HttpWatcher *WatchClient
)
var WatchThreadNum = 4;
var ticker = time.NewTicker(time.Duration(outTime) * time.Second) // --- A
type MessageWatcher struct {
	start  int
	offset int
}

type WatchClient struct {
	url string
	lineHttpChan chan *MessageWatcher
}

var ch = make(chan *gohttp.AsyncResponse)

func NewHttpWatcher( address string) (afk *WatchClient, err error) {
	afk = &WatchClient{
		url:address,
		lineHttpChan:make(chan *MessageWatcher,10000),
	}
	if err != nil {
		fmt.Printf("Failed to create Connetcion: %s\n", err)
		os.Exit(1)
	}
	for i:=0;i<WatchThreadNum;i++{
		// 根据配置文件循环开启线程去发消息到kafka
		go afk.Watcher(ch)
	}
	go  PusherKafka(ch)
	return
}


func InitHttpWatcher()(err error){
	var url = fmt.Sprintf("https://shive.cn/api/sql/query?token=%s&project=%s", "123456", "hive-aliyun")
	HttpWatcher,err = NewHttpWatcher(url)
	return
}

func (k *WatchClient) Watcher(chan *gohttp.AsyncResponse) {

	//从channel中读取日志内容放到kafka消息队列中
	logs.Info("[start watcher]")
	req := gohttp.NewRequest()


	for v := range k.lineHttpChan {
		var formdata =  map[string]string{}
		formdata["q"] = fmt.Sprintf("SELECT * FROM events order by time  limit  %d,%d", v.start, v.offset)
		logs.Info(formdata["q"])
		formdata["format"] = "json"
		var headerVals  =  map[string]string{}

		req.FormData(formdata).Headers(headerVals).AsyncPost(k.url, ch)
	}

}

func (k *WatchClient) AddTask(s int, e int) (err error) {
	logs.Info("Http Add Message")
	k.lineHttpChan <- &MessageWatcher{start: s, offset: e}
	return
}

func PusherKafka(ch chan *gohttp.AsyncResponse){
	for i:=range ch{
		var result ,_ = i.Resp.GetBodyAsString();
		if result=="" && i.Resp==nil{

			HttpWatcher.AddTask((start) * MiniBatch,MiniBatch)
			logs.Warn("wait ...")
		}else {
			logs.Info(result)
		}
	}
}

func main() {
	InitHttpWatcher()
	loopWorker()
}

func loopWorker(){
	defer ticker.Stop()
	for {
		select {
		case <-ticker.C:
			// 执行我们想要的操作
			start ++;
			HttpWatcher.AddTask((start) * MiniBatch,MiniBatch)
		}
	}
}


```

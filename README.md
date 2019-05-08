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
package main

import (
	"bufio"
	"fmt"
	"github.com/astaxie/beego/logs"
	"github.com/nahid/gohttp"
	"github.com/rs/xid"
	"io"
	"os"
	"strings"
	"time"
)
var MiniBatch = 256;
var outTime int64 = 9;
var start = 0;

var (
	HttpWatcher *WatchClient
)
var WatchThreadNum = 1;
var ticker = time.NewTicker(time.Duration(outTime) * time.Second) // --- A
type MessageWatcher struct {
	start  int
	offset int
}

type WatchClient struct {
	url string
	lineHttpChan chan *MessageWatcher
}

var chHttp = make(chan *gohttp.AsyncResponse,100)

func NewHttpWatcher( address string) (afk *WatchClient, err error) {
	afk = &WatchClient{
		lineHttpChan:make(chan *MessageWatcher,10),
	}
	if err != nil {
		fmt.Printf("Failed to create Connetcion: %s\n", err)
		os.Exit(1)
	}
	afk.url  = address;
	for i:=0;i<WatchThreadNum;i++{
		// 根据配置文件循环开启线程去发消息到kafka
		go PusherKafka()
		go afk.Watcher()
	}

	return
}


func InitHttpWatcher()(err error){
	var url = fmt.Sprintf("https://2as1.cn/api/sql/query?token=%s&project=%s", "21t1rxxxxx", "default")
	HttpWatcher,err = NewHttpWatcher(url)
	return
}




func (k *WatchClient) Watcher() {

	//从channel中读取日志内容放到kafka消息队列中
	logs.Info("[start watcher]")
	req := gohttp.NewRequest()
	for v := range k.lineHttpChan {
		var formdata =  map[string]string{}
		formdata["q"] = fmt.Sprintf("SELECT * FROM events order by time  limit  %d,%d", v.start, v.offset)
		logs.Info(formdata["q"])

		formdata["format"] = "json"
		var headerVals  =  map[string]string{}

		req.FormData(formdata).Headers(headerVals).AsyncPost(k.url, chHttp)
	}
}

func (k *WatchClient) AddTask(s int, e int) (err error) {
	k.lineHttpChan <- &MessageWatcher{start: s, offset: e}
	return
}
func PusherKafka(){
	for i:=range chHttp{

		if i.Resp.GetResp().StatusCode!=200 || i.Resp.GetResp().ContentLength==0 {
			logs.Info("wait .....")
			continue
		}else {
			var result ,_ = i.Resp.GetBodyAsString()

			rd := bufio.NewReader(strings.NewReader(result))
			// 有数据进行++ 没有数据进行watch on
			start ++;
			for {
				line, err := rd.ReadString('\n')
				if err != nil || err == io.EOF {
					break
				}
				line = strings.Replace(line, "\f", "", -1)

				// 执行我们想要的操作
				var newLine = fmt.Sprintf(",\"properties_uuid\":\"%s\"}",GetUuid())
				var temp = strings.Replace(line,"}",newLine,strings.Index(line,"}"))
				fmt.Printf("the line %s",temp)
				HttpSender.addMessage(temp,"buried-point-mall")
			}
		}
	}
}

func GetUuid()string{
	//h := md5.New()
	//h.Write(data)
	//s := hex.EncodeToString(h.Sum(nil))
	//fmt.Println(s)
	guid := xid.New()
	return fmt.Sprintf("000000000000%s",guid.String())
}


func main() {
	InitHttpWatcher()
	InitHttpPusher(true)
	loopWorker()
}

func loopWorker(){
	defer ticker.Stop()
	for {
		select {
		case <-ticker.C:
			HttpWatcher.AddTask( start * MiniBatch,MiniBatch)
		}
	}
}
```

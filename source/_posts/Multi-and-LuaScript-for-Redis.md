---
title: Multi and LuaScript for Redis
date: 2019-03-16 14:03:05
tags:
    - Redis
    - Lua
---
## Multi? What is it?
主要執行multi和exec包圍起來的部分，當multi命令發出，redis會進入transaction狀態，redis會進入blocking，不再處理其他請求，直到發出multi的session發出exec命令為止。
被multi和exec包圍起來的命令們進入獨享redis的過程，直到執行完成。
因為是transction，所以命令要全部執行完畢，不然就是都不執行。
如果exec命令送出前，client斷線，redis會清空transction queue，所有命令都不會執行。
一但client送出了exec命令，所有命列就會被執行，就算client斷線了也無訪。

如果transction過程中，要執行3個命令 1、2、3，其中2出錯了，不會像db那樣整個rollback，依然會執行到完。

透過這種方式，redis就能避免多個client同時訪問，出現讀寫不一致的情況，來完成atomic transction操作。
![](/images/Redis/TbzqlS7.png)

還有DISCARD (取消transcation)、WATCH(監控某個KEY，只要被更動，則transction無法被觸發，exec會得到nil)
![](/images/Redis/Qqmv4m2.png)
由於Multi是把命令逐條發送給redis server，server還會回應QUEUED，並且最後還要回應執行結果，所以封包數量上其實比平常都多，效率也近乎最低的。

## Pipeline?
一次執行多條命令，無關atomic，網路封包數量也最少。
有機會再筆記。

## Why use Lua Script to access Redis?
* 當需要對redis下多個命令，且每一個命令就是一次網路傳輸。
* 多個指令中，後面的指令依賴前一個操作的結果時。
* Redis依然是 單執行緒下執行依序執行這些操作。
* 如果Lua script本身內容很多，可以先把lua script載入redis, redis會返回一組SHA字串，以後就直接傳遞這SHA字串即可替代原內容。
* 可以組合多個命令，且該次執行本身也是atomic操作。
* 支援base、[table](http://huli.logdown.com/posts/198866-lua-table)(array)、string、match、debug、[cjson](https://www.kyne.com.au/~mark/software/lua-cjson-manual.html)、[cmspack](https://github.com/antirez/lua-cmsgpack)。

```
情境 :
HotYoutubers 是以sorted set結構存放，
檢查要是youtuber不再名單內，則新增
要是在名單內了，則score + 1
```

Initial Data
```javascript
const youtubers = ['理科太太', '赤井Akai', 'D Rebound 99', '融融歷險記', '志祺七七X圖文不符', '閃亮胖時代','只會玩刀鋒', 'Ken桑', '尬酒螺仔', '我們Our channel'];
const voteCnt = [10000, 9999, 8999, 8000, 8001, 9383, 5345, 6864, 1384, 5131];
const newYoutubers = ['华农兄弟'];
const key = "HotRanks";
const youtuberListKey = "Youtubers";
const promiseArray: any[]  = [];
youtubers.map(d => promiseArray.push(d));
await Promise.all(youtubers.map(d) => {
    redisClient.SADD("KEYA", d);
});
```

**不使用Multi**
```javascript
client.zscan(key, "0" , "MATCH", youtubers[0], (err, reply) => {
    if(err){
        console.error({
            error: err,
            key: key,
            target:  youtubers[0]
        })
        return ;
    }
    if(reply[1].length > 0) {
        console.log(`${youtubers[0]} increase score`);
        client.zincrby(key, 1, youtubers[0]);
    } else {
        console.log("add new youtuber");
        client.zadd(key, 1, youtubers[0]);
    }
});
```

網路封包 : 5個封包
![](/images/Redis/VDOhrRC.png)

**使用Multi**
```javascript
    client
    .multi()
    .zscan(key, "0" , "MATCH", youtubers[0])
    .zincrby(key, 1, youtubers[0])
    .zrevrange(key, 0 , 10)
    .exec((err, replies) => {
        if(err){
            console.error({
                error: err,
                key: key,
                target:  youtubers[0]
            })
            return ;
        }
        console.dir(replies);
        if(replies[0][1].length > 0) {
            console.log(`${youtubers[0]} increase score`);
        } else {
            console.log("add new youtuber");
        }
    })
結果 :
[ [ '0', [ '理科太太', '10007' ] ],
  '10008',
  [ '理科太太',
    '赤井Akai',
    '閃亮胖時代',
    'D Rebound 99',
    '志祺七七X圖文不符',
    '融融歷險記',
    'Ken桑',
    '只會玩刀鋒',
    '我們Our channel',
    '尬酒螺仔' ] ]
理科太太 increase score
```

封包數量 : 3個, 一個是multi起transaction,並把命令們丟進去queue，等到exec被發出調用，一次返回全部命令的結果。
![](/images/Redis/x21LwjK.png)  

**But!!!**
不方便做到更複雜的需求!
雖然zincrby在item不存在時，會幫忙新增item，並給上分數。
但要是想先檢查youtube set內內是否存在此youtuber時就很難了。

* 使用Lua Script 來完成!
    * 先檢查youtuber清單 “Youtubers”
    * youtuber存在，則增加分數
    * youtuber不存在，則回傳nil

Notes :
`Lua的array都是從1開始的。
client.eval(luaScript, 2, key, youtuberListKey, newYoutubers[0]; 這段的2是告訴redis有兩個Key在KEYS[]當中，而在這所引外的都會是在ARGV[]當中了。
宣告變數是用local這關鍵字宣告
lua的null是nill
如果變數x是table(即arry)類型，要使用 則使用#x 來使用; 例如取得x的陣列長度 #x.length
想要對table類型做歷尋有以下方式
使用ipair探索table中的陣列部分, for k, v in ipair(變數x) do ; k就是k, v則是value, ipair
使用pairs探索table中所有資料, for k, v in ipair(變數x) do ; k就是k, v則是value, ipair`


```javascript
const luaScript = 'local youtuber = redis.call("HEXISTS", KEYS[2], ARGV[1]) \
if(youtuber == 0) then \
  return nil \
else \
  return redis.call("zincrby", KEYS[1], 1, ARGV[1]) \
end ';
client.eval(luaScript, 2, key, youtuberListKey, newYoutubers[0], (err, replies) => {
            if(err){
                console.error({
                    error: err,
                    key: key,
                    target:  youtubers[0]
                })
                return ;
            }
            console.dir(replies);
        })
// 結果 : null
client.eval(luaScript, 2, key, youtuberListKey, youtubers[0], (err, replies) => {
            if(err){
                console.error({
                    error: err,
                    key: key,
                    target:  youtubers[0]
                })
                return ;
            }
            console.dir(replies);
        })
// 結果 : '10010'
```

封包數量 : 3, 同Multi
![](/images/Redis/j6fwoVP.png)


*But!!!*
每次執行都要送這些腳本以及編譯，有沒有方法省掉呢?
Yes!!!
先把script透過script load載入，會得到一串hash string。
以後執行evalsha跟這hash string即可。
```javascript
let hashScript: string;
client.script('load', luaScript, (err, res) => {
    console.dir(res);
    // 'aa838cb2f4f84408889222a7af3bec845f126ba8'
    hashScript = res;
})
client.evalsha(hashScript, 2, key, youtuberListKey, youtubers[0], (err, replies) => {
            if(err){
                console.error({
                    error: err,
                    key: key,
                    target:  youtubers[0]
                })
                return ;
            }
            console.dir(replies);
        })
```

Redis能做到的事情蠻多的，不只是能當快取，透過Lua腳本，也能簡單的做些關聯查詢。
只是它畢竟是單執行緒，要是被這任務卡住太久，就喪失快取的意義了。

日後有機會再筆記PUB/SUB，我在實務上的簡單應用。
最主要的還是他的叢集架設與資料同步/備份的部分。
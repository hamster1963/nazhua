# Nazhua
基于哪吒监控(nezha.wiki)v0版本构建的前端主题，目前兼容与v0相同数据结构的v1版本。  
主题有点**重**，因为内置了一个带中文的`SarasaTermSC-SemiBold`字体。  

## 劝退指南 用前必读
1. 本主题是基于哪吒监控v0版本构建的，不确定能否完美v1版本。*20241206的版本已适配*  
2. 本主题是一个纯前端项目，需要解决跨域问题，通常需要一个nginx或者caddy反代请求解决跨域问题。  
3. 我不会提供任何技术支持，如果你有问题，可以提issue，但是我不保证会回答，可能询问GPT会更快。  

## V0/V1的使用区别  
V1必须在`config.js`中指定`nezhaVersion`的版本为`v1`，**大小写敏感*  
默认的数据是基于V0

## 关于点阵地图  
点阵地图是一个失真的地图，地图边际与城市位置都不是真实的经纬度坐标，因此无法通过经纬度来定位城市。  
需要在是[Nazhua配置生成器](https://hi2shark.github.io/nazhua-generator/)中，拾取点阵地图上的坐标，然后在`config.js`中配置`customCodeMap`来自定义地图点信息。  
如何指定节点的地理位置？  
在哪吒监控后台，给节点的公开备注对象中，添加一个`customData`对象，并指定`location`的代码；  
关于都有哪些内置的地理位置代码，需要在[Nazhua配置生成器](https://hi2shark.github.io/nazhua-generator/)中查看。  
示例
```json
{
  "customData": {
    "location": "HKG"
  }
}
```
对于几个我常见的国别位置，添加了默认映射位置，会自动显示在地图上。（美国太大了，就默认显示在最常买的位置：洛杉矶）

## 关于节点slogan和购买链接
同时，这个`customData`中还可以添加一项`slogan`和`orderLink`字符串，分别用于显示节点的标语和购买链接。
```json
{
  "customData": {
    "location": "HKG",
    "slogan": "这是一个香港节点",
    "orderLink": "https://buy.example.com"
  }
}
```
Tips:   
由于配置数据获取的方式特殊，无法正常解析符号`&`，建议在[https://www.bejson.com/enc/urlencode/](https://www.bejson.com/enc/urlencode/)进行编码后，将encodeURIComponent编码内容添加到orderLink中。  
当然，你也可以通过浏览器的console（控制台），执行`encodeURIComponent('链接内容')`，获取编码后的内容。  

## 对于公开备注的支持
在哪吒的主题ServerStatus迭代中，nap0o增加了一个公开备注的功能，可以给节点添加额外的展示信息  
具体字段定义参考 [https://github.com/nezhahq/nezha/pull/425](https://github.com/nezhahq/nezha/pull/425)  
Nazhua对这个支持大概在90%左右，参与数据处理了的字段如下：  
```json
{
   "billingDataMod": {
       "startDate": "2024-10-01T00:00:00+08:00",
       "endDate": "2024-11-01T00:00:00+08:00",
       "autoRenewal": "1",
       "cycle": "月",
       "amount": "$3.99"
   },
   "planDataMod": {
       "bandwidth": "30Mbps",
       "trafficVol": "1TB/月",
       "trafficType": "1",
       "networkRoute": "CN2,GIA",
       "extra": "传家宝,AS9929"
   }
}
```
其中IPv4、IPv6暂未参与到处理中，后续可能会支持。  

## 数据来源
1-0. 公开的全量配置，其中包括“公开备注”（PublicNote），来着探针主页上暴露的服务器节点列表配置信息。此处是根据正则匹配的方式，获取到的节点列表。在主题项目中，默认将访问`/nezha/`的指向此处。  
2-0. 实时数据，v0来着公开的ws服务接口，`/ws`。  
2-1. 实时数据/全量数据，v1来着公开的ws服务接口，`/api/v1/ws/server`。  
3-0. 监控数据，v0来着公开的api接口，`/api/v1/monitor/${id}`。  
3-1. 监控数据，v1来着公开的api接口，`/api/v1/service/${id}`。  
4-0. 分组数据，v0来着服务器节点列表的`Tag`字段匹配。  
4-1. 分组数据，v1来着公开的api接口，`/api/v1/server-group`。  

## 部署
Nazhua主题是一个纯前端项目，可以部署在纯静态服务器上，但需要解决`/api/v1/monitor/${id}`监控数据、`/ws`WS服务和`/`主页的跨域访问。  
通常来说，你需要一个nginx或者caddy反代请求解决跨域问题。  

### Docker Compose + Cloudflare Tunnels部署
**请关注备注中的提示内容**
```yaml
services:
  nazhua:
    image: ghcr.io/hi2shark/nazhua:latest
    container_name: nazhua
    ports:
      - 80:80
    # volumes:
      # - ./favicon.ico:/home/wwwroot/html/favicon.ico:ro # 自定义favicon图标
      # - ./config.js:/home/wwwroot/html/config.js:ro # 自定义配置文件
      # - ./style.css:/home/wwwroot/html/style.css:ro # 自定义样式文件
    environment:
      - DOMAIN=_ # 监听的域名，默认为_（监听所有）
      - NEZHA=http://nezha-dashboard.example.com/ # 可以被反代nezha主页地址
    restart: unless-stopped
```
建议通过docker-compose部署服务，然后通过cloudflare的tunnels向外提供服务，可以不用自己配置https证书。  
**再次提示，哪吒V1的必须设置config.js中的nezhaVersion为v1**  

### Nginx配置示例
```nginx
server {
  listen 80;
  server_name nazhua.example.com;
  client_max_body_size 1024m;

  # 哪吒V0的WebSocket服务
  location /ws {
    proxy_pass ${NEZHA}ws;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }

  # 哪吒V1的WebSocket服务
  location /api/v1/ws/server {
    proxy_pass ${NEZHA}api/v1/ws/server;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }

  location /api {
    proxy_pass http://nezha-dashboard.example.com/api;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }

  location /nezha/ {
    proxy_pass http://nezha-dashboard.example.com/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }

  location / {
    try_files $uri $uri/ /index.html;
    root /home/wwwroot/html;
  }
}
```


## 自定义配置
可以通过修改根目录下的`config.js`文件来自定义配置  
例如：(*参考内容在文档上不一定是最新，具体参考public/config.js或者[Nazhua配置生成器](https://hi2shark.github.io/nazhua-generator/)*)
```javascript
window.$$nazhuaConfig = {
  title: '哪吒监控', // 网站标题
  freeAmount: '白嫖', // 免费服务的费用名称
  infinityCycle: '无限', // 无限周期名称
  buyBtnText: '购买', // 购买按钮文案
  listServerStatusType: 'progress', // 服务器状态类型--列表
  detailServerStatusType: 'progress', // 服务器状态类型--详情页
  hideNavbarServerCount: false, // 隐藏服务器数量
  hideNavbarServerStat: false, // 隐藏服务器统计
  hideListItemStatusDonut: true, // 隐藏列表项的饼图
  hideListItemStat: true, // 隐藏列表项的统计信息
  hideListItemBill: false, // 隐藏列表项的账单信息
  hideWorldMap: false, // 隐藏地图
  hideHomeWorldMap: false, // 隐藏首页地图
  hideDetailWorldMap: false, // 隐藏详情地图
  hideFilter: false, // 隐藏筛选
  hideTag: false, // 隐藏标签
  customCodeMap: {}, // 自定义的地图点信息
  nezhaVersion: 'v0', // 哪吒版本
  apiMonitorPath: '/api/v1/monitor/{id}', // 监控数据API路径，不明白的就不要改
  wsPath: '/ws', // WebSocket路径，不明白的就不要改
  nezhaPath: '/nezha/', // 哪吒主页路径，不明白的就不要改
  nezhaV0ConfigType: 'servers', // 哪吒v0数据读取类型
  v1ApiMonitorPath: '/api/v1/service/{id}', // v1监控数据API路径，不明白的就不要改
  v1WsPath: '/api/v1/ws/server', // v1WebSocket路径，不明白的就不要改
  v1GroupPath: '/api/v1/server-group', // v1分组数据API路径，不明白的就不要改
  routeMode: 'h5', // 路由模式
};
```
可以通过修改根目录下的`style.css`文件来自定义样式  
例如：
```css
:root {
  /* 修改颜色 */
  /* 地图上标记点的颜色 */
  --world-map-point-color: #fff;
  /* 列表项显示的价格颜色 */
  --list-item-price-color: #ff6;
  /* 购买链接的主要颜色 */
  --list-item-buy-link-color: #f00;
}
```
可以通过[Nazhua配置生成器](https://hi2shark.github.io/nazhua-generator/)快速生成config.js配置文件


## 二次开发提示
`.env.development.local`配置变量
```bash
WS_HOST=http://127.0.0.1:9288 # 本地nezha ws反代
API_HOST=http://nezha-dashboard.example.com # 本地nezha api反代
NEZHA_HOST=http://nezha-dashboard.example.com # 本地nezha主页反代
NEZHA_HOST_REPACE_PATH=1 # 是否替换主页路径`/nezha/`
```

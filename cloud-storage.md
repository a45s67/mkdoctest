---
share: true
---
## S3
- 除錯 b2 s3 api 回傳 signature not match 的原因
	- 從這篇可知 [disscussion](https://stackoverflow.com/questions/40856473/signature-does-not-match-403-error-when-signing-a-url-via-aws-sdk-go)，送出去的任何設定可能都是會被驗證的 header 的一部分，檢查一遍權限，發現 `ALLOW_LIST_BUCKET` 權限沒對齊
		- 改正之後就沒問題了
-
- 研究圖片有時候會 404 的原因
	- cf 上有好幾個類似的提問，但都沒人回
	- https://saturncloud.io/blog/cloudflare-images-missing-with-404-error-only-on-chrome/
	- 找不到原因，尤其是重整之後圖片都仔入夠快了...
		- 開無痕試可以成功測出載入失敗
	- 測試一遍後，bucket、application key 的設定都沒有錯，用工具都抓的到圖片，推測是腳本問題
		- 看 b2 的 sdk 範例，其實短短幾行就可以送了
	- 參考文件調整腳本
		- 推測是使用者傳過去時包含的 header 太髒...，我直接把一堆不需要送的給拿掉，只送 http method 和 url 就 ok 了。
		- https://github.com/mhart/aws4fetch


## 價格比較
  - aws s3
    - 流量極貴
  - b2
    - 容量便宜 $5/TB
    - 跟 cf 接流量免費
    - 取名麻煩，不能跟其他人撞名
  - r2
    - 容量價格還行 $15/TB
    - 流量免費，請求要算錢，每月 100w 筆免費
- 建一個 r2 storage & s3 api worker 50m


## 使用 juicefs 掛載 docker volume

### 作法
方法一: 使用 docker plugin (不 work)
- 無法跟 docker composer 一起使用，如下寫法，啟動時會卡住 ，因為  volume 會先初始化，但這時 redis 還沒啟動，導致 jfsvolume 啟動失敗
``` yaml
version: "1.0"

name: juicefs-demo

services:
  busybox:
    image: busybox
    depends_on:
      - redis
    command: "ls /jfs"
    volumes:
      - jfsvolume:/jfs
  
  redis: # 
    image: redis/redis-stack:latest
    volumes:
      - redis-data:/data

volumes:
  jfsvolume:
    driver: juicedata/juicefs
    driver_opts:
      name: cdn4
      metaurl: redis://redis:6379 
      storage: s3
      bucket: https://f0bXXXXX.r2.cloudflarestorage.com/danbooru
      access-key: 88aXXXXX
      secret-key: 610eXXXXXXXX
  redis-data:

```

方法二: 使用 docker service 來 mount volume，這招 work
``` yaml
version: "1.0"

name: juicefs-demo

services:
  juicefs:
    image: juicedata/mount:ce-v1.1.0
    # Error: failed to open /dev/fuse: Operation not permitted
    # https://github.com/tencentyun/cosfs/issues/26: 启动容器的时候，可以加一下--privileged参数
    # [How to bring up a docker-compose container as privileged?](https://stackoverflow.com/questions/69052575/how-to-bring-up-a-docker-compose-container-as-privileged)
    privileged: true
    depends_on:
      - redis
      - postgres
    volumes:
      - jfs-mount:/mnt/jfs
      - postgres-data:/var/lib/postgresql/data
    # [Docker Compose - How to execute multiple commands?](https://stackoverflow.com/questions/30063907/docker-compose-how-to-execute-multiple-commands)
    command: >
      bash -c "
      juicefs format --bucket $OBJECT_URL --access-key $ACCESS_KEY --secret-key $SECRET_KEY --storage s3 $META_URL $MOUNT_NAME;
      juicefs mount $META_URL /mnt/jfs"

  postgres:
    image: evazion/postgres
    environment:
      POSTGRES_USER: juicefs
      POSTGRES_HOST_AUTH_METHOD: trust
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  jfs-mount:
  postgres-data:
```
.env
```
MOUNT_NAME=cdn4
META_URL=postgres://juicefs@postgres/juicefs
OBJECT_URL=https://f0b1XXX.r2.cloudflarestorage.com/danbooru
ACCESS_KEY=88a8XXXX
SECRET_KEY=610eXXXXX
```

### 成功後的樣子
![[juicefs-demo.png|juicefs-demo.png]]
如上圖，從 mount 的資料夾下新增 123 和 test/2.txt，在 s3 中會以 chunk 形式儲存，跟想像中不一樣...，並且把檔案直接複製到 s3 時，mount fs 是不會同步這一個檔案的。

# 建置 MQTT Client，並使用 MQTT 控制硬體執行 FW 更新

## 前言

由於公司的多功能底座產品所匹配使用的網路硬碟產品停產了，為了使單獨的底座也可以獨自運作，我使用 MQTT 並建立排成腳本(crontab)定期發送裝置相關訊息透過 MQTT 協議至與底座綁定的雲端伺服器，並透過前端網頁顯示資料，而為了做到控制底座，必須在底座上新增 MQTT Client 接收資料，訂閱固定的**主題**(Topic)，並分析**訊息**(Message)內容，來完成指定的動作。

## MQTT Client

網路上其實很多有關於 MQTT Client 的程式，但多半只有程式，因為執行的載體不同，導致編譯的工具也會有所不同，以下程式為 MQTT Client 的主程式。

```c++
//收到MQTT訊息的Callback
void my_message_callback(struct mosquitto *mosq, void *userdata, const struct mosquitto_message *message)
{
	if (message->payloadlen)
	{
		printf("topic:%s msg:%s\n", message->topic, message->payload);
	}
	else
	{
		printf("%s (null)\n", message->topic);
	}

	fflush(stdout);
}

//成功連線Server的Callback
void my_connect_callback(struct mosquitto *mosq, void *userdata, int result)
{
	int i;
	if (!result)
	{
		//成功連線後訂閱指定的主題
		mosquitto_subscribe(mosq, NULL, "CMD/#", 2);
	}
	else
	{
		fprintf(stderr, "Connect failed\n");
	}
}

//訂閱成功的Callback
void my_subscribe_callback(struct mosquitto *mosq, void *userdata, int mid, int qos_count, const int *granted_qos)
{
	int i;

	printf("Subscribed (mid: %d): %d", mid, granted_qos[0]);
	for (i = 1; i < qos_count; i++)
	{
		printf(", %d", granted_qos[i]);
	}
	printf("\n");
}

// 主程式
int main(int argc, char *argv[])
{
	bool clean_session = true;
	char cmdStr[500];
	char IP[25];
    //讀取綁定的Server IP
	snprintf(cmdStr, 500, "sqlite3 %s \'Select Server_IP from MQTTInfo\'", dbPath);
	sendCMD(cmdStr, IP);

	struct mosquitto *mosq = NULL;

	mosquitto_lib_init();
	mosq = mosquitto_new(NULL, clean_session, NULL);
	if (!mosq)
	{
		fprintf(stderr, "Error: Out of memory.\n");
		return;
	}
    //設定指定的function為Callback
	mosquitto_connect_callback_set(mosq, my_connect_callback);
	mosquitto_message_callback_set(mosq, my_message_callback);
	mosquitto_subscribe_callback_set(mosq, my_subscribe_callback);

    //MQTT連線至綁定的Server
	mosquitto_connect(mosq, IP, 1883, 600);

	mosquitto_loop_forever(mosq, -1, 1);

	mosquitto_destroy(mosq);
	mosquitto_lib_cleanup();
}
	return 0;
}
```

為了達到控制的目的，接收到訊息後可分析訊息內容，達到控制目的，由於綁定於伺服器的裝置有許多台，為了辨別此訊息的接收對象是否為接收到訊息的裝置，可透過訂閱的主題或在訊息中夾帶裝置資訊來進行過濾，訊息內容唯一個 Json 字串，用於判定要執行哪些操作。

```c++
void my_message_callback(struct mosquitto *mosq, void *userdata, const struct mosquitto_message *message)
{
	char *SplitDel = "/";
	char *topic_class[2];
    //讀取裝置的Mac address作為判斷依據
	char mac[50];
	sendCMD("cat /etc/config/network  | awk -F \"'\" 'NR==15 {print $2}'", mac);
	strtok(mac, "\n");
	if (message->payloadlen)
	{
		split(topic_class, message->topic, SplitDel);
		if (strcmp(*(topic_class + 1), mac) == 0)
		{
			char *value = message->payload;
            //字串轉Json物件
			cJSON *msg = cJSON_Parse(value);
			int cmd = cJSON_GetObjectItem(msg, "cmd")->valueint;
			switch (cmd)
			{
			case FW_UPGRADE:
			{
				//執行用Python寫的FW更新腳本
                system("python /etc/mosquitto/FirmwareUpdate.py > /dev/null 2>&1 &");
			}
			break;
			case DELETE_FILE:
			{
				//...
			}
			break;
			default:
				printf("Do Nothing\n");
				break;
			}
		}
		printf("topic:%s ID:%s msg:%s\n", *(topic_class + 0), *(topic_class + 1), message->payload);
	}
	else
	{
		printf("%s (null)\n", message->topic);
	}
	fflush(stdout);
}
```

韌體更新的流程使用 Python 撰寫，由於將 FW 的 Image 檔放於 AWS 上，下載連結為動態產生而不是固定連結，因此要使用 HTTP Request 取得下載 Image 檔的連結，而 C++沒有可直接使用的 Library，透過 C++觸發 python 腳本執行 FW 更新。

```python
#取得目前最新的FW資訊
def get_firmware_info():
    try:
        ssl._create_default_https_context = ssl._create_unverified_context
        connectUrl = httplib.HTTPSConnection(
            "app.transcendcloud.com", timeout=10)
        reqHeader = {"Authorization": "x-transcend-server",
                     "Content-Type": "application/json",
                     "Accept": "application/json"}
        data = json.dumps({"tag": "firmware",
                           "category": "DPB",
                           "target": "DPD6N"})

        connectUrl.request("POST", "/v1/api/get/version",
                           data, reqHeader)

        res = connectUrl.getresponse()
        jsonData = json.load(res)
        print(jsonData)
        return jsonData["result"]
    except Exception as e:
        send_mqtt({"Status": "Network Error"})
        sys.exit("Get Firmware info fail")

#取得下載FW的URL
def get_firmware_download_url(path):
    try:
        connectUrl = httplib.HTTPSConnection(
            "app.transcendcloud.com", timeout=10)
        reqHeader = {"Authorization": "x-transcend-server",
                     "Content-Type": "application/json",
                     "Accept": "application/json"}
        data = json.dumps({"action": "getObject",
                           "tag": "firmware",
                           "category": "DPB",
                           "path": path,
                           "target": "DPD6N"})

        connectUrl.request(
            "POST", "/v1/api/generateLink", data, reqHeader)

        res = connectUrl.getresponse()
        jsonData = json.load(res)
        print(jsonData)
        return jsonData["result"]["url"]
    except Exception as e:
        send_mqtt({"Status": "Network Error"})
        sys.exit("Get Firmware download url fail")

#下載檔案至/tmp
def download(url):
    filedata = urllib2.urlopen(url)
    datatowrite = filedata.read()
    with open('/tmp/install.img', 'wb') as f:
        f.write(datatowrite)

#確認MD5是否正確
def doChecksum(md5):
    download_md5 = os.popen('md5sum /tmp/install.img').read().split(" ")[0]
    print(md5, download_md5)
    print("Download Finish")
    return str(md5).upper() == download_md5.upper()

#執行FW更新
def fw_update():
    os.popen('echo true > /etc/mosquitto/var/updateStatus')
    send_mqtt({"Status": "Rebooting"})
    os.popen('/sbin/sysupgrade /tmp/install.img')

#使用MQTT與前端網頁溝通，將目前更新狀態傳送至前端顯示
def send_mqtt(data):
    server_ip = os.popen(
        'sqlite3 /home/database.db \'select Server_IP from MQTTInfo\'').readlines()[0].replace("\n", "")
    mac = os.popen(
        'cat /etc/config/network  | awk -F \"\'\" \'NR==15 {print $2}\'').readlines()[0].replace("\n", "")
    data["Mac"] = mac
    cmd = '/usr/bin/mosquitto_pub -h ' + \
        server_ip + ' -t "Status/Firmware/Update" -m "' + \
        json.dumps(data).replace('"', '\\"') + '"'
    cmd = cmd.encode('utf-8')
    os.popen(cmd)
    print(cmd)

#主程式開始
data = get_firmware_info()
url = get_firmware_download_url(data["path"])
send_mqtt({"Status": "Downloading"})
download(url)
status = doChecksum(data["checksum"])
print(status)
if status:
    fw_update()
else:
    send_mqtt({"Status": "Download Fail"})
    sys.exit("Download Fail")
```

### 編譯

由於要將 MQTT Client 於多功能底座(`OpenWRT`)上執行，需透過交叉編譯器(`Cross compiler`)將程式編譯為執行檔。

- 交互編譯器路徑: SDK/staging_dir/toolchain-arm_cortex-a7+neon_gcc-4.9-linaro_glibc-2.19_eabi/bin/arm-openwrt-linux-gcc

編譯的命令格式如下，編譯完成後可將執行檔移至裝置上測試功能

```
//[交互編譯器路徑] [程式檔案(.c)] -o [輸出檔案名稱] -L [引用Library路徑] -I [引用標頭檔路徑] [所有使用到的Library]

$ SDK/staging_dir/toolchain-arm_cortex-a7+neon_gcc-4.9-linaro_glibc-2.19_eabi/bin/arm-openwrt-linux-gcc main.c json_lib/cJSON.c -o mqtt_client -L Lib/ -I Include/ -lmosquitto -lpthread -lcares
```

> 標頭檔與 Library 需使用`make menuconfig`將`mosquitto`啟用後，先行編譯，於 SDK/build_dir/target-arm_cortex-a7+neon_glibc-2.19_eabi/mosquitto-nossl/mosquitto-1.4.10/lib 路徑下，將所有.h 檔複製至`Include`資料夾內，並且將 Library 檔(`.a`,`.so`)複製至`Lib`資料夾內。

### 包裝映像檔(Package image)

OpenWRT 的基本架構含有底層的 Linux 原始碼和 Packages，為了使 MQTT Client 於裝置開機時就執行，因此要修改 Linux 原始碼中的`rc.local`檔，其路徑於 SDK/package/base-files/files/etc/rc.local，將以下程式加入。

```bash
#啟用MQTT Client
/etc/mosquitto/mqtt_client &
```

而後，將有關 MQTT 的程式放置於 /SDK/realtek/packages/net/mosquitto/files 的路徑下，修改 mosquitto 資料夾內的`Makefile`將編譯的流程加入 Packaging 流程中，程式如下。

```makefile
define Package/mosquitto/install/default
	#...此部分為原始編譯mosquitto執行檔的流程...
    #以下為新增的部分
	$(TARGET_CROSS)gcc -o ./files/myClient/mqtt_client ./files/myClient/main.c ./files/myClient/json_lib/cJSON.c -L ./files/myClient/Lib/ -I ./files/myClient/Include/ -lmosquitto -lpthread -lcares
	$(INSTALL_BIN) ./files/myClient/mqtt_client $(1)/etc/mosquitto/
	$(CP) ./files/script $(1)/etc/mosquitto/
	$(CP) ./files/myClient/FirmwareUpdate.py $(1)/etc/mosquitto/
    #為了避免Debug時，沒有權限建立Log檔，因此先預先建立後複製到指定路徑下
    $(INSTALL_DIR) $(1)/home
    $(CP) ./files/myClient/testLog $(1)/home
endef
```

## 其他優化

由於 MQTT Client 是透過連線至指定 Server 的方式建立連線，若使用者在使用的過程中更改了綁定的伺服器，則需要將 MQTT Client 的 Process kill 掉重新建立連線，因此需要在觸發變更 IP 時執行以下程式。

```c++
// Restart mqtt client
char pid[20];
sendCMD("pidof /etc/mosquitto/mqtt_client", pid);
strtok(pid, "\n");
snprintf(cmdStr, 500, "kill %s", pid);
system(cmdStr);
system("/etc/mosquitto/mqtt_client &");
```

- 此程式位於/SDK/realtek/packages/net/lighttpd/files/mqtt-api/mqtt.c 內，此檔案是編譯使用者連線至多功能底座的本地端網頁時使用的 Web API，當使用者透過本地端網頁觸發變更綁定 Server IP 時，執行讀取 PID、Kill PID，並重新啟動 MQTT Client process。

# **訊息通報 - 申請與串接 Line Notify 、 Slack 教學**
### 客製化訊息通報模式，並串接常用通訊軟體 Line 與 Slack 等等 ...，利於開發者隨時隨地監控程式狀況，降低開發人員大量重工、低效率撰寫程式。

## **使用流程規劃**
1. 數個專案需被監控，如排程性 爬蟲專案、預測服務、網站服務
2. 監控分為「系統狀態」與「異常訊息」兩大項目並各司其職
3. 「系統狀態」必需監控主程式；「異常訊息」則必需監控其呼叫各子函式，並即時終止運行。
4. 取得訊息立即通報目標通訊軟體
5. 定時整合性通知 - 兩項目依時間統整
![Concept](/images/concept.drawio.png)

## **任務項目 - 依上述需求安排以下工作**
- [X] Line Notify - 申請 Line Notify 流程並取得 Token
- [X] Slack - 申請 Slack API 流程並取得 Token
- [X] Config - 建立 ini 檔案存放相關資訊
- [X] Preprocessing - 建立物件其具備自動 Retry GET/POST Requests 工具
- [X] Preprocessing - 讀取 config.ini 檔案與 Import 所需套件
- [X] Preprocessing - 建置物件與主要控制函式
- [X] Line Notify - 建立次要函式，其主要利用 Token 發送訊息
- [X] Slack - 建立次要函式，利用 Token 發送訊息、檔案至目標 channel 與指定人員
- [X] Slack - 取得目標 Token 中 Users 與 Channels 資訊
- [X] Example - 使用範例與成果
- [ ] Summary Reports - 整合、統計 Slack 訊息發送至 Line Notify
- [ ] Package - 包裝成 wheel 格式至 PyPi-Server 利於反覆使用

## **Line Notify - 申請 Line Notify 流程並取得 Token**
1. 登入 [Line Notify 官方網站](https://notify-bot.line.me/zh_TW/)

2. 右上角下拉式窗 - 進入「個人頁面」

    ![個人頁面](/images/LINE_02.png)

3. 進入頁面會看到「已連動的服務」與「發行存取權杖(開發人員用)」 

    ![已連動服務](/images/LINE_03.png)

    ![發行存取權杖](/images/LINE_04.png)

4. 選擇「發行權杖」並設定權杖名稱與指定聊天室(群組或自己1對1)

    ![設定權杖資訊](/images/LINE_05.png)

    ![複製權杖Token](/images/LINE_06.png)

## **Slack - 申請 Slack API 流程並取得 Token**
1. 登入 [Slack API 官方網站](https://api.slack.com/)

2. 選擇 Customize your workspace 中 Create an app

    ![Slack API 官網](/images/Slack_01.png)

3. 選擇 Create New App > From scratch > 輸入 App 名稱 與 選擇 Workspace

    ![建置 App](/images/Slack_02.png)

    ![輸入 App 基本資訊](/images/Slack_03.png)

4. 建置 App 成功網頁轉至 基本資訊頁面 > 需設定 Incoming Webhooks、Permissions

    ![設定權杖資訊](/images/Slack_04.png)

5. 開啟 Incoming Webhooks 設定

    ![開啟 Incoming Webhooks](/images/Slack_05.png)
6. 安裝 App 到指定 Workspace

    ![建置 Token](/images/Slack_06.png)

    ![選擇 App 安裝頻道](/images/Slack_07.png)

    ![產生 Token](/images/Slack_08.png)

7. 設定 Permission 中 App 可使用功能

    ![增加權限畫面](/images/Slack_09.png)

    ![設定權限功能](/images/Slack_10.png)

8. 重新啟動 App 

    ![重新啟動](/images/Slack_11.png)

## **Config - 建立 config.ini 檔案存放相關資訊**
```ini
[Common]
# 取得 Token
TOKEN_LINE_NOTIFY = XXX 
TOKEN_SLACK = XXX

URL_LINE_NOTIFY = https://notify-api.line.me/api/notify # Line Notify API Url

ENABLE_NOTIFY_LINE = True # 控制是否通報 Line Notify
ENABLE_NOTIFY_SLACK = True # 控制是否通報 Slack
```

## **讀取 config.ini 檔案與 Import 所需套件**
```python
import os, traceback, requests, urllib3
import pandas as pd
from dotenv import load_dotenv # pip install python-dotenv
from slack_sdk import WebClient # pip install slack-sdk

assert load_dotenv('config.ini') == True
```

## **Package - 建立物件其具備自動 Retry GET/POST Requests 工具**
```python
class Crawler:
    
    def __init__(self, CONN_RETRY_COUNT=4, CONN_STATUS_FORCELIST=[429, 500, 502, 503, 504]):
        
        self.CONN_RETRY_COUNT = CONN_RETRY_COUNT
        self.CONN_STATUS_FORCELIST = CONN_STATUS_FORCELIST
        
    
    def __conn__(self):
        
        session = requests.Session()
        adapter = requests.adapters.HTTPAdapter(max_retries=urllib3.Retry(total=self.CONN_RETRY_COUNT, backoff_factor=1, allowed_methods=None, status_forcelist=self.CONN_STATUS_FORCELIST))
        session.mount("http://", adapter)
        session.mount("https://", adapter)
        
        return session
    
    def __send__(self, method, url, verify=False, timeout=5, params={}, headers={}, **kwargs):
        
        session = self.__conn__()
        
        if method == 'GET':
            response = session.get(url=url, headers=headers, verify=verify, timeout=timeout, params=params)
        elif method == 'POST':
            data = kwargs['data']
            response = session.post(url=url, headers=headers, verify=verify, timeout=timeout, data=data)

        if not 'response' in locals():
            return -999, 'ERROR'
        
        return response.status_code, response

```

## **建置物件與主要控制函式**
```python
class Notify:

    TOKEN_LINE_NOTIFY = os.environ['TOKEN_LINE_NOTIFY']
    TOKEN_SLACK = os.environ['TOKEN_SLACK']

    URL_LINE_NOTIFY = os.environ['URL_LINE_NOTIFY']

    ENABLE_NOTIFY_LINE = eval(os.environ['ENABLE_NOTIFY_LINE'])
    ENABLE_NOTIFY_SLACK = eval(os.environ['ENABLE_NOTIFY_SLACK'])

    # 確保通知相關參數有被宣告
    def __init__(self):
        
        assert self.ENABLE_NOTIFY_LINE != None
        assert self.ENABLE_NOTIFY_SLACK != None

    # 作為發送訊息主控台 - 呼叫各通訊軟體
    def __message__(self, title:str, msg:str, **kwargs):
        if self.ENABLE_NOTIFY_LINE == True:
            self.__line__(msg='{} - {}'.format(title, msg))
        
        if self.ENABLE_NOTIFY_SLACK == True and 'channel' in kwargs:
            tags, channel = '', kwargs['channel']
            
            msg = "*{}*\n{}".format(title, msg)
            if 'members' in kwargs:
                if len(kwargs['members']) >= 1:
                    tags = str(['<@{}>'.format(user_id) for user_id in kwargs['members']]).replace("['", '').replace("']", '')
                    msg += '\n{}'.format(tags)
            
            self.__slack__(msg=msg, channel=channel, mrkdwn=True, is_file=False)

        if self.ENABLE_NOTIFY_LINE == False and self.ENABLE_NOTIFY_SLACK == False:
            print(
                'Display Message - {}\n\t{}'.format(title, msg)
            )

    # 作為發送檔案主控台 - 呼叫 Slack 傳送檔案
    def __fileUpload__(self, title:str, msg:str, channel:str, filename:str):

        if os.path.isfile(filename):
            msg = "*{}*\n{}".format(title, msg)
            self.__slack__(msg=msg, channel=channel, mrkdwn=True, is_file=True, filename=filename)
        else:
            print('[Notify][Slack] - File Not Exist')

```
## **Line Notify - 建立次要函式，其主要利用 Token 發送訊息**
```python
    # 發送訊息至 Line Notify
    def __line__(self, msg:str):
        assert self.TOKEN_LINE_NOTIFY != None

        headers = {
            "Authorization": "Bearer " + self.TOKEN_LINE_NOTIFY, 
            "Content-Type" : "application/x-www-form-urlencoded"
        }

        data = {'message': msg}

        status_code, response = Crawler().__send__(method='POST', url=self.URL_LINE_NOTIFY, verify=True, headers=headers, data=data)

        if status_code == 200:
            print('[Notify][Line Notify] - Success')
        else:
            print('[Notify][Line Notify] - Fail')
```

## **Slack - 建立次要函式，利用 Token 發送訊息、檔案至目標 channel 與指定人員**
```python
    # 發送 訊息/檔案 至 Slack
    def __slack__(self, msg:str, channel:str, is_file:bool, mrkdwn:bool, filename=None):
        assert self.TOKEN_SLACK != None
        
        client = WebClient(token=self.TOKEN_SLACK)

        if is_file == False:
            response = client.chat_postMessage(channel=channel, text=msg, mrkdwn=mrkdwn)
        elif is_file == True and os.path.isfile(filename):
            response = client.files_upload_v2(
                channel=channel,
                file=filename,
                initial_comment=msg
            )

        if 'response' in locals():
            status_code = response.get('ok')

            if status_code == True:
                print('[Notify][Slack] - Success')
            else:
                print('[Notify][Slack] - Fail')
                
        else:
            print('[Notify][Slack] - Send Fail')
```

## **Slack - 取得目標 Token 中 Users 與 Channels 資訊**
```python
class SlackInfo:
    
    TOKEN_SLACK = os.environ['TOKEN_SLACK']
    
    def __init__(self):
        self.client = self.__conn__()
        
    def __conn__(self):
        return WebClient(
            token=self.TOKEN_SLACK, 
            # ssl=sslcert, e.g. from ssl import SSLContext; sslcert = SSLContext()
            # proxy=proxyinfo # e.g. "http://localhost:9000"
        )
            
    def get_channels_list(self):
        return self.client.conversations_list()['channels']
    
    def get_users_list(self):
        return self.client.users_list()['members']

```

## **Handler - 用裝飾詞 Decorator 監控各函式的運作** 
```python
class Handler:
    def __init__(self, type_name:str, source_name:str, channel:str, member:str):
        self.type_name = type_name
        self.source_name = source_name
        self.channel = channel
        self.member = member
        
    def __call__(self, func):
        def wrapper(*args, **kwargs):

            try:
                func(*args, **kwargs)
                Notify().__controller__(title='[{}][{}]'.format(self.type_name, self.source_name), msg='Finish', channel=self.channel, members=self.member)

            except Exception as e:
                Notify().__controller__(title='[{}][{}]'.format(self.type_name, self.source_name), msg=traceback.format_exc(), channel=self.channel, members=self.member)

        return wrapper
```
## **Example**
```python
# Slack - Get Channels and Members
channels = pd.DataFrame(SlackInfo().get_channels_list())
members = pd.DataFrame(SlackInfo().get_users_list())
 
# Example 1. Notify Status
Notify().__controller__(title='[Example 1 - Notify Status]', msg='<@ron.jl.lee>', channel=os.environ['SLACK_CHANNEL'])

# Example 2. Notify Exception
try:
    c
except Exception as e:
    Notify().__controller__(title='[Example 2 - Notify Exception]', msg=traceback.format_exc(), channel=os.environ['SLACK_CHANNEL'])

# Example 3. Upload File to Slack
Notify().__fileUpload__(title='[Example 3 - Upload File to Slack]', msg='Upload file', channel=os.environ['SLACK_CHANNEL'], filename='README.md')

# Example 4. Combine Decorator 
@Handler(type_name='INFO', source_name='Example 4 - Combine Decorator', channel=os.environ['SLACK_CHANNEL'], member=[])
def f(a, b):
    print('calling f with args ', (a, b))
f(a=1, b=2)

# Example 5. Combine Decorator
@Handler(type_name='INFO', source_name='Example 5 - Combine Decorator', channel=os.environ['SLACK_CHANNEL'], member=[])
def f(a, b):
    print('calling f with args ', (a, b, c))
f(a=1, b=2)

```
### **結果呈現**
![結果呈現 - LINE  ](/images/RES_LINE.png)
![結果呈現 - SLACK ](/images/RES_Slack.png)
### 以上簡易教學文章供參考，若有其餘改善建議或常用需求可以提出討論。
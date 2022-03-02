# PYTHON-台灣即時股價與資料庫串接
運用台灣證券交易（http://mis.twse.com.tw ） 所提供之API，查詢即時報價，並透過Python與SSMS串接，將資料匯入資料庫中，以便後續分析利用。完整程式可參考get_stock_price.ipynb
首先套件所需模組：
```python
from IPython.display import display, clear_output
from urllib.request import urlopen
import numpy as np
import pandas as pd
import datetime
import requests
import sched
import time
import json
import twstock
import pyodbc
import schedule
import sys
```

## 查詢即時股價
以list的方式產生股票代碼，可支援多檔股票查詢，targets中以台積電（2330）和聯發科（2454）為例，接著產生stock_list為後續API所需的格式，具體格式為，'http://mis.twse.com.tw/stock/api/getStockInfo.jsp?ex_ch=tse_2330.tw|tse_2454.tw' ，街個透過json模組將資料讀取。

```python
    #  stock_list
    targets = ['2330','2454']
    stock_list = '|'.join('tse_{}.tw'.format(target) for target in targets) 

    #　query data
    query_url = "http://mis.twse.com.tw/stock/api/getStockInfo.jsp?ex_ch="+ stock_list
    data = json.loads(urlopen(query_url).read())
```

## 拆解資料
http://mis.twse.com.tw/stock/api/getStockInfo.jsp?ex_ch=tse_2330.tw|tse_2454.tw 中的JSON格式資料，股價資料在msgArray裡，內層資訊可參考下方表格，

|Name	|Description|
| :-----| :---- |
|tlong|	epoch毫秒數|
|f	|揭示賣量(配合「a」，以_分隔資料)|
|ex	|上市別(上市:tse，上櫃:otc，空白:已下市或下櫃)|
|g	|揭示買量(配合「b」，以_分隔資料)|
|d	|最近交易日期(YYYYMMDD)|
|b	|揭示買價(從高到低，以_分隔資料)|
|c	|股票代號|
|a	|揭示賣價(從低到高，以_分隔資料)|
|n	|公司簡稱|
|o	|開盤|
|l	|最低|
|h	|最高|
|w	|跌停價|
|v	|累積成交量|
|u	|漲停價|
|t	|最近成交時刻(HH:MM:SS)|
|tv	|當盤成交量|
|nf	|公司全名|
|z	|當盤成交價|
|y	|昨收|

表格資料參考自:https://zys-notes.blogspot.com/2020/01/api.html

揭示買價、量跟賣價、量的五檔部分以'_'區分，在此先將買價、量跟賣價、量第一檔擷取出來
```python
    df0 = pd.DataFrame(data['msgArray'], columns=['f','g','b','a'])
    df1 = pd.DataFrame({
        'bid_volumn':df0['f'].str.split('_', expand=True)[0],
        'ask_volumn':df0['g'].str.split('_', expand=True)[0],     
        'bid_price':df0['b'].str.split('_', expand=True)[0],
        'ask_price':df0['a'].str.split('_', expand=True)[0],
                 })
```                 

接著將依些資訊抓取出來，如：時間、股票代碼、公司名稱、現在價格、交易數量、累積數量、開盤價、等資訊。最後將買價、量跟賣價、量第一檔合併為一個資料集。

```python
    columns = ['t','c','n','z','tv','v','o','h','l','y']
    df = pd.DataFrame(data['msgArray'], columns=columns)
    df.columns = ['curr_time','tickers','namec','current_price','current_volumn','acc_volumn','s_open','hight','low','y_close']
    df = pd.concat([df, df1], axis=1)
```     
另外資料本身為字串模式，若沒抓到股價則回傳'-'，因此在處理上先將該筆刪除，而後將股價轉換為小數點後兩位的數字，而交易數量等資訊轉換為整數。最後再增加今天日期。

```python
    df = df[df['current_price']!='-']
    for i in ['current_price','s_open','hight','low','y_close','bid_price','ask_price','bid_price1','ask_price1']:
        df[i] = df[i].apply(lambda x: round(float(x),2))
    for i in ['current_volumn','acc_volumn', 'bid_volumn','ask_volumn', 'bid_volumn1','ask_volumn1']:
        df[i] = df[i].apply(lambda x: int(x))
    # 增加日期
    df['date'] = datetime.datetime.now().date()
```    
### 包裝成Function
```python
def stock_realtime_fn(targets):
    #today date 
    time = datetime.datetime.now().date()

    #stock_list
    stock_list = '|'.join('tse_{}.tw'.format(target) for target in targets) 

    #　query data
    query_url = "http://mis.twse.com.tw/stock/api/getStockInfo.jsp?ex_ch="+ stock_list
    data = json.loads(urlopen(query_url).read())
#
    # get data
    columns = ['t','c','n','z','tv','v','o','h','l','y']
    df = pd.DataFrame(data['msgArray'], columns=columns)
    df0 = pd.DataFrame(data['msgArray'], columns=['f','g','b','a'])
    df1 = pd.DataFrame({
        'bid_volumn':df0['f'].str.split('_', expand=True)[0],
        'ask_volumn':df0['g'].str.split('_', expand=True)[0],     
        'bid_price':df0['b'].str.split('_', expand=True)[0],
        'ask_price':df0['a'].str.split('_', expand=True)[0],
        'bid_volumn1':df0['f'].str.split('_', expand=True)[1],
        'ask_volumn1':df0['g'].str.split('_', expand=True)[1],     
        'bid_price1':df0['b'].str.split('_', expand=True)[1],
        'ask_price1':df0['a'].str.split('_', expand=True)[1]  
                 })
    df.columns = ['curr_time','tickers','namec','current_price','current_volumn','acc_volumn','s_open','hight','low','y_close']
    df = pd.concat([df, df1], axis=1)
    df = df[df['current_price']!='-']
    for i in ['current_price','s_open','hight','low','y_close','bid_price','ask_price','bid_price1','ask_price1']:
        df[i] = df[i].apply(lambda x: round(float(x),2))
    for i in ['current_volumn','acc_volumn', 'bid_volumn','ask_volumn', 'bid_volumn1','ask_volumn1']:
        df[i] = df[i].apply(lambda x: int(x))
    # 增加時間
    df['date'] = time
    # 紀錄更新時間
    time0 = datetime.datetime.now()
    #print(df)
    return df
```
## 串接資料庫與更新資料
本文章使用SSMS資料庫，要用到pyodbc模組，首要再SSMS上創建好資料庫以及使用者名稱和密碼，並建立連線，在此不透過WINNDOWS連線是透過sql sever驗證的方式。詳細使用可參考：
https://docs.microsoft.com/zh-tw/sql/machine-learning/data-exploration/python-dataframe-sql-server?view=sql-server-ver15
```python
    # connection    
    server = '{YOUR_SEVER_NAME}' 
    database = '{YOUR_database}' 
    username = '{YOUR_username}' 
    password = '{YOUR_password}' 
    cnxn = pyodbc.connect('DRIVER={SQL Server};SERVER='+server+';DATABASE='+database+';UID='+username+';PWD='+ password)
``` 
### 建立資料表
首先判斷'stock_realtime_v2'資料表是否已經存在，若不存在則創建資料表。資料表的格式與待會要輸入資料的格式要一致，要先了解欄位為數字或文字的形式，包含整數或小數等。
``` python
data_table_name='stock_realtime_v2'
index = True
    if index == True :
        index = False
        sql = "IF NOT EXISTS (SELECT * FROM sysobjects WHERE name='"+data_table_name +"' AND xtype='U')"+'CREATE TABLE '+data_table_name+''' ( 
                    date nvarchar(16),
                    curr_time nvarchar(10), 
                    tickers nvarchar(10), 
                    name nvarchar(10), 
                    current_price NUMERIC(10,2), 
                    current_volumn int,
                    bid_volumn int,
                    ask_volumn int, 
                    bid_price NUMERIC(10,2), 
                    ask_price NUMERIC(10,2), 
                    acc_volumn int, 
                    s_open NUMERIC(10,2), 
                    hight NUMERIC(10,2), 
                    low NUMERIC(10,2), 
                    y_close NUMERIC(10,2)
                    )
        '''
        # excute
        cursor = cnxn.cursor()
        cursor.execute(sql)
        cnxn.commit()

``` 
接著將上一階段產生的資料插入'stock_realtime_v2'資料集中，架構上為一筆筆用INSERT INTO插入資料表中，即可完成串接資料庫並匯入。

``` python
st_list= ['2454','2330']
data = stock_realtime_fn(st_list)
# Insert Dataframe into SQL Server:
    sql = "INSERT INTO " +data_table_name +'''
     (date, curr_time, tickers, name, current_price, current_volumn, 
     bid_volumn, ask_volumn, bid_price, ask_price, 
     acc_volumn, s_open, hight, low, y_close) values(?,?,?,?,?,?
     ,?,?,?,?
     ,?,?,?,?,?) 
    '''
    cursor = cnxn.cursor()
    for index, row in data.iterrows():
         cursor.execute(sql,
                        str(row.date), row.curr_time, row.tickers,row.namec, row.current_price, row.current_volumn, 
                        row.bid_volumn, row.ask_volumn , row.bid_price, row.ask_price,
                        row.acc_volumn, row.s_open,  row.hight,  row.low,  
                        row.y_close)        

    cnxn.commit()
    cursor.close()
```
## 週期性任務設定
上述串接資料庫的方式為執行一次，會將當筆的資料增加至資料表中，但台灣股市是9:00-13:30為開盤時間，避免要手動一直執行的情況，在此設定週期性任務的方式，讓電腦在特定時間執行程式，
首先設定兩個函數，get_stock_exec為上述資料庫串接的方式，輸入為要創建的資料表名稱，與股票代號；另一個函數為exit，為中止函數。
``` python
def get_stock_exec(data_table_name,st_list= ['2454','2330']):
    data = stock_realtime_fn(st_list)
    # connection    
    server = '{YOUR_SEVER_NAME}' 
    database = '{YOUR_database}' 
    username = '{YOUR_username}' 
    password = '{YOUR_password}' 
    cnxn = pyodbc.connect('DRIVER={SQL Server};SERVER='+server+';DATABASE='+database+';UID='+username+';PWD='+ password)
    # create table
    index = True
    if index == True :
        index = False
        sql = "IF NOT EXISTS (SELECT * FROM sysobjects WHERE name='"+data_table_name +"' AND xtype='U')"+'CREATE TABLE '+data_table_name+''' ( 
                    date nvarchar(16),
                    curr_time nvarchar(10), 
                    tickers nvarchar(10), 
                    name nvarchar(10), 
                    current_price NUMERIC(10,2), 
                    current_volumn int,
                    bid_volumn int,
                    ask_volumn int, 
                    bid_price NUMERIC(10,2), 
                    ask_price NUMERIC(10,2), 
                    bid_volumn1 int, 
                    ask_volumn1 int,
                    bid_price1 NUMERIC(10,2), 
                    ask_price1 NUMERIC(10,2),
                    acc_volumn int, 
                    s_open NUMERIC(10,2), 
                    hight NUMERIC(10,2), 
                    low NUMERIC(10,2), 
                    y_close NUMERIC(10,2)
                    )
        '''
        # excute1
        cursor = cnxn.cursor()
        cursor.execute(sql)
        cnxn.commit()
        
    cursor = cnxn.cursor()
    # Insert Dataframe into SQL Server:
    sql = "INSERT INTO " +data_table_name +'''
     (date, curr_time, tickers, name, current_price, current_volumn, 
     bid_volumn, ask_volumn, bid_price, ask_price, 
     acc_volumn, s_open, hight, low, y_close) values(?,?,?,?,?,?
     ,?,?,?,?
     ,?,?,?,?,?) 
    '''
     
    for index, row in data.iterrows():
         cursor.execute(sql,
                        str(row.date), row.curr_time, row.tickers,row.namec, row.current_price, row.current_volumn, 
                        row.bid_volumn, row.ask_volumn , row.bid_price, row.ask_price,
                        row.acc_volumn, row.s_open,  row.hight,  row.low,  
                        row.y_close)        

    cnxn.commit()
    cursor.close()
    
def exit():
    print('Now the system will exit ') 
    sys.exit()
```

接著設定執行的條件或是方式，在此以每兩秒鐘執行一次讀取股價的方式，因仿間傳聞5秒內讀取超過2次會被封鎖，而在收盤時間一點半則會執行終止程式。

```python
schedule.every(2).seconds.do(get_stock_exec,data_table_name='stock_realtime_v2',st_list= ['2454','2330'])   
schedule.every().day.at('13:31:00').do(exit)

while True:
    schedule.run_pending()
```

若順利執行完成會看到以下的結果，代表主執行緒中退出。
```
An exception has occurred, use %tb to see the full traceback.

SystemExit
```
若要重新執行則需用以下程式碼，將所有排成設定先清除，再重新跑一遍即可。
```python
schedule.clear()
```
## 後記
目前測試執行上有發現，讀取股價時無法抓取到所有即時的資訊，會遺失一些交易資料，有時候一分鐘都沒資料進來，或是有抓到重複的資料，應該是資料本身還沒更新到，該資料未來會使用在當沖策略上的，之後會研究券商的API，看能不能有更及時的資料，因目前所知的如yahoo finance只能抓到15分鐘前的延遲報價資料，沒辦法取得當下資料，但該資料可以抓到7天內短時間的交易資訊，之後有時間也會把yahoo finance範例補齊。



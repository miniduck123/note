# 談談Multiple Dispatch與fixture的化學反應
## 所以那個Overload呢
在說到multiple dispatch以前，要先來了解“python是沒有overlaod”這件事。
簡單前情提要一下所謂的overlaod指的是，檔案內容中有兩個相同命名的function，但是接收參數的形態不同或者數量不同。
某些語言支持overlaod的寫法，可以自行判斷兩個function是不同的，根據呼叫函數所傳送的參數去找到對應的function去執行。
舉個c++例子：
```cpp
int area(int length, int breadth) {
  return length * breadth;
}

float area(int radius) {
  return 3.14 * radius * radius;
}

```

而如果用python做類似的定義方式去執行的時候，我們不難發現一個現象：
```python
def area(leng,breadth):#直接被完全無視的一個funciton
    return length * breadth;
def area(radius):
    return 3.14 * radius * radius;

```

python只會呼叫後面定義的function，編譯的時候基本上也不會拋錯。對python來說後定義的function自然而然就是覆寫掉前面定義的相同function。
###### 看似很炫炮結果python直接華麗轉身的略過了（笑


如果需要尋求python不支援overlaod的原因，可以參考如下翻譯文章：
https://iter01.com/605349.html
裡面還有一段重要的引文：
> **如果使用 functools 庫的 singledispatch 裝飾器，Python 也可以實現函式過載。**

## Multiple Dispatch
雖然singledispatch裝飾器也是可以辦到overlaod的效果，不過這邊介紹一下multiple dispatch。
###### 主要是我覺得用起來相對直觀簡單好懂XDD
要使用multipledispatch首先要透過pip安裝
```
pip install multipledispatch
```
安裝回來以後就可以import進去想要使用的檔案內：

```python
from multipledispatch import dispatch

@dispatch(int,int)#根據兩個數值參數放進去dispatch的修飾詞內
def area(leng,breadth):
    return length * breadth;

@dispatch(int)#根據單個數值參數放進去dispatch的修飾詞內
def area(radius):
    return 3.14 * radius * radius;

@dispatch(str)#改變參數型態
def area(area_name):
    print(area_name);
```
添加@dispatch來區分多個function其實不同，呼叫函數的時候會根據dispatch修飾詞去判斷呼叫哪一個函數。
### 一定要“load”一下才好嗎
其實python可以動態設定function要用多少參數的狀況來說，不見得要這樣寫。
會出現想要function的名稱一樣的狀況，往往是因為function其實功能一樣，但是可能會有需要動態傳入額外的參數做其他判斷的時候。

舉一個例子：我需要get很多隻api的response。最簡單的做法就是塞入url就好。
```python
def get_response_status(url):
		response = requests.get(url,headers=self.headers,verify=False)
		status = response.status_code
		if status == 200 :
			return True
		else:
			return False

```
但是有時候我們總會需要傳入一些payload或者data等等
```python
#兩個函數做的事情一樣，但就是差了一個data的參數，所以我要命名不一樣的function name
def get_response_status(url):
		response = requests.get(url,headers=self.headers,verify=False)
		status = response.status_code
		if status == 200 :
			return True
		else:
			return False
        
def get_response(url,data):
		response = requests.get(url,headers=self.headers,verify=False,data=data)
		status = response.status_code
		if status == 200 :
			return True
		else:
			return False
```
當我們需要的不同參數的時候，get_response可能就會演化各種各樣的：get_response_json()、get_response_data()、．．．．．．。
##### 我真的覺得我的記性無法記住到底有幾種get_response
### 動態數量參數
承接上一部分的例子的code。當我們不確定今天這個函數會需要多少個參數的時候，可以動態設定function的可接參數：
```python
def get_response_status(*args):
    for arg in args:
        print(arg)

get_response_status('https://google.com',['I', 'have', 'a', 'data'])
```
![](https://i.imgur.com/eXKJH7j.png)

不過如果傳送值的順序錯誤的時候，很可能會發生接錯值的問題:
```python
def get_response_status(*args):
    for arg in args:
        print(arg)

get_response_status(['I', 'have', 'a', 'data'],'https://google.com')
```
還有如果今天狀況是兩個都是單個參數，但是傳送的值型態不一樣：
```python
def get_response_status(data):
   #要先對data預處理，判別型態等等等等等．．．．

get_response_status('susan test')
get_response_status(666)
```
以及不同參數組合：
```python
def get_response_status(data):
   #要先對data預處理，判別型態等等等等等．．．．

get_response_status('susan test')
get_response_status(666)
get_response_status('susan test',666)
get_response_status(666,['susan','jerlad','vita'])
```

以上這幾種狀況都需要先在呼叫的function當中寫下大量的預處理，去判斷寫入的參數是不是自己要的。畢竟難免會有不小心送錯參數的時候:)
即便使用dict的型別去包裝所有的值，也要去判斷哪些key可能沒有value等等。
相較來說運用@dispatch修飾詞，把本來動態的function強制變成靜態的，就可避免掉很多這類的困擾。
```python
#限制了只能傳入怎樣型態的參數以及預期傳入的個數，也可以限制第幾個參數應該是什麼型別。
@dispatch(str)
def get_response_status(url):
		response = requests.get(url,headers=self.headers,verify=False)
		status = response.status_code
		if status == 200 :
			return True
		else:
			return False
@dispatch(str,object)        
def get_response_status(url,data):
		response = requests.get(url,headers=self.headers,verify=False,data=data)
		status = response.status_code
		if status == 200 :
			return True
		else:
			return False
```
## MultipleDispatch + fixture
筆者在研究pytest的fixture的時候，發現了一個很微妙的小應用。
我在某些共用fixture當中想要透過判斷某些狀況以後，再去yield對應的driver之類的值。
比如：站點可能有登入跟不登入的狀態要測試，可能可以根據要不要傳入test_account來改變yeild的driver。
```python
@pytest.fixture(scope="module")
def page_route():
	driver = webdriver.Chrome()
    
    @dispatch(str,str)
	def _page_route(page_name,domain):#我不需要登入的時候跑這個
		if page_name == "susan_index":
			driver.get(f"https://url?site={domain}")
		elif page_name == "susan_game":
			driver.get(f"https://url?site={domain}")
		
		
		return driver

    @dispatch(str,str,str)
	def _page_route(page_name,domain,login_account):#我需要登入的時候跑這個
		if page_name == "susan_index":
			driver.get(f"https://url?site={domain}")
		elif page_name == "susan_game":
			driver.get(f"https://url?site={domain}")
	    
        #do something about login..........
		
		return driver
	yield _page_route
	driver.close()
```
至於testcase的部分要怎麼去呼叫這個複雜的fixture?
```python
def test_without_login(page_route,domain_param):
	game_index= page_route("susan_index",domain_param.get('domain'))
    
def test_with_login(page_route,account_param,domain_param):
	game_index= page_route("susan_game",account_param,domain_param.get('domain'))
```
基本上透過fixture的code，因為yield回傳的是_page_route這個function，所以testcase接到的東西就從以往習慣的web driver變成一個function。
這時候把他當成一個普通的function去帶值即可。

#### 但是筆者覺得也可以拆成兩個fixture互相呼叫就可以了XD
```python
@pytest.fixture(scope='module')
def driver(request):
	driver = web(page_url)
	yield driver
	driver.close()


@pytest.fixture(scope='module',params = test_account)#要登入吃這個
def game_page_login(driver,request):	
        driver.login_flow()#走個登入流程
	driver.send_accounts(request.param)
	driver.get(page_url)
	yield {'driver':driver,'test_acocunt':request.param}
	driver.clean_cookies()
	driver.refresh()
	

	
@pytest.fixture(scope='module')#不登入吃這個
def game_page_logout(driver,request):
	driver.get(page_url)
	yield {'driver':driver}
	
```
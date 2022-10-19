# 關於yield & fixture
### fixture修飾詞
使用pytest進行自動化測試，常常會使用到fixture修飾詞。這個修飾詞集合了setup／teardown的功能。
也就是說，可以直接使用單一個修飾詞配合一個function，就可以達到一些測試進行前或者之後的動作。
```python
@pytest.fixture(scope='module')
def driver(request):
	driver = web(page_url)
	yield driver#先等等，我馬上回來
	driver.close()#事情都處理完了，我回來做這一行code
```
如上面的示範code。這個動作會在所有測試進行之前，先建立一個新的driver物件出來，然後，中間yield他，最後做了一個driver關閉的動作。
因為每一次的測試步驟起始都是“打開瀏覽器”，然後最後動作是“關閉瀏覽器”。
每一次跑testcase都要做這些重複的動作，就可以包進去fixture裡面。pytest就會自行判斷這些動作要先進行或者最後進行。
### yield v.s. return
那麼為什麼function之中，不是使用return?
大部分的function在執行後，如果需要回傳值，會習慣使用return。
但是在fixture裏面較少會使用。
return代表了，執行到這個地方，回傳參數值以後，就不會再回來function做後續的程序了。
```python
@pytest.fixture(scope='module')
def driver(request):
	driver = web(page_url)
	return driver#掰掰我不會再回來囉
	driver.close()#被遺棄的一行code
```
上述內容，我把yield改成return。
此時執行的時候會發現，當所有測試執行完畢後，driver不會進行關閉動作。因為return以後，程序將不會再返回這個function做driver.close()
但是yield可以再回傳這個值以後，自動記住做完其他程序動作，要記得回來這個function，把後續的程序執行完畢。
## 在fixture裡面夾帶參數的應用
#### 參數模組化測試
我們知道pytest有另外提供一個參數模組化的功能。
當需要測試不同數據組合狀況下，功能是否都正常，可以使用參數設定的方式進行測試。
```python
@pytest.mark.parametrize(
	"date_option",['本週','上週','本月','上月']
)
@pytest.mark.parametrize(
	"user_option",["全部顯示"]
)
def test_report_by_date_option_and_source(date_option,user_option):
	
	assert True
```
假設今天有下拉選單給予使用者選擇日期區間，不太可能根據每一個下拉單選項各自寫一次testcase。
應用parametrize這個修飾詞可以丟入所需要的參數，pytest會自動去區分“有多少參數我就帶入參數跑多少次這條testcase”

雖然說這可以應用在很多條testcase當中，可是有一個很大的問題。
“假如今天所有case都要共用同樣的參數呢？？？“
### 參數也一起fixture吧！
當今天的案例，有兩條case，一個專門進行使用者登入，另一個則是使用者登入後使用其他功能。
此時會需要“同一個測試帳號”去跑完這兩條case。
如果按照上述code的範本，則每一個case都要記得去引用所有參數，而且這兩條case會分開執行。
也許今天可以換個思維，把這個參數放進去fixture，讓他放入各個testcase當中執行。
```python
@pytest.fixture(scope='function',params = test_account)
```
根據pytest官方文件的說明，我們知道fixture這個修飾詞裡面有包含一個可用的設定參數。
叫做params。
顧名思義就是攜帶參數值進去的意思。
如此一來只要“使用這個fixture的function，就一定可以直接拿到fixture設定的參數內容“
至於如何使用它？
```python
test_account=['susan','vita','jerlad']
@pytest.fixture(scope='function',params = test_account)
def game_page(driver,request):
	driver.login_flow()
	driver.send_accounts(request.param)
	driver.get(page_url)
	yield {'driver':driver,'test_acocunt':request.param}
	
```
只要在function裏面，記得加上request，然後呼叫這個request的param，就可以逐一把參數引用進去。
###### !  params可以是一list型態的或者其他型態的數據組。pytest會自動逐一去取得數據內容。 !
以上述示範為例，我取得測試帳號，先進行登入流程，然後跳轉到功能頁面後，我因為還需要帳號相關的資訊進行後續測試動作，所以我把他跟driver一起yield出去給不同testcase使用。
### 進階參數化設定－yield又yield
既然可以在fixture的function中使用yield，當然在普通的function中也可以使用！
舉個例子，假設我今天“需要取得待測網站上的所有網址，進行api調用”
假設先有一個api物件。
```python
class api_object():
	def __init__(self,url):#先把呼叫api的一些預設值產生好
		self.headers={'Content-Type':'application/json; charset=utf-8'}	
		self.page_url = url
        
    def get_response_status(self,url):#一個取得status code的func
		response = requests.get(url,headers=self.headers,verify=False)
		status = response.status_code
		if status == 200 :
			return True
		else:
			return False
    
    def get_all_urls(self):#取得當前頁面所有url
		result = get_html_response(self.page_url)
		links_href = result.find_all(href=True)
        return links_href

```
然後呢我們的testcase的文件內容可能這樣寫：
```python
from api_object import *

api_obj = api_object(page_url)#進行測試的網址塞進去產生api物件
all_apis = api_obj.get_all_urls()#取回所有頁面上的url

@pytest.mark.parametrize(
	"api",all_apis
)
def test_url_status(api):
    for api in api:
        ###keep going testing ###
        assert api_obj.get_response_status(api)#迴圈尋訪物件內容
	
    
```
然後api內容可能要自行進行迴圈尋訪一次次的進行get_response_status()。
雖然不是不可以這樣撰寫，不過pytest有一個功能，就是帶入的參數，會自動顯示在testcase名稱上，方便後續測試管理知道，哪一個參數帶進去後，造成fail的結果。
如果沒有把參數逐一引入testcase的話，pytest只會顯示參數物件本身的名稱，重複的名稱自動加上序號，不會顯示是哪一條url。
![](https://i.imgur.com/jBCwEvL.png)
###### 能猜中哪個url是哪個算你厲害囉！
如果今天要達成把參數分次逐一引入，那麼勢必“return回來的就是單獨一條一條的網址“
我們修改一下api物件當中回傳url的function內容：
```python
class api_object():
	def __init__(self,url):#先把呼叫api的一些預設值產生好
		self.headers={'Content-Type':'application/json; charset=utf-8'}	
		self.page_url = url
        
    def get_response_status(self,url):#一個取得status code的func
		response = requests.get(url,headers=self.headers,verify=False)
		status = response.status_code
		if status == 200 :
			return True
		else:
			return False
    
    def get_all_urls(self):#取得當前頁面所有url
		result = get_html_response(self.page_url)
		links_href = result.find_all(href=True)

		for link_h in links_href:
			url_txt = link_h.get('href')
			if url_txt.startswith('https'):
				yield url_txt

```
根據前面我們了解到yield跟return的不同，我改寫了這個function，也就是說迴圈當中他會逐一把網址丟回去呼叫這個function的“人”
丟完之後，會再回來迴圈內執行，再把下一次尋訪到的url在丟回去。直到迴圈結束。

那麼這個function做好了以後，testcase可以怎麼改接值？
```python
from api_object import *

api_obj = api_object(page_url)

@pytest.fixture(scope='module',params= api_obj.get_all_urls())
#承接前面我們學到的方式，直接呼叫回傳url的物件function，塞進params
def api(request):
	yield request.param

def test_url_status(api):#驗證status正常200
	assert api_obj.get_response_status(api)
    
def test_url...(api):#如果還要針對這個url進行其他驗證的testcase
    #keep going  to test other case
```
此時跑完測試後，會發現pytest會顯示每一個url進行status code驗證後的對應結果
方便管理哪一個url現在狀態異常。
![](https://i.imgur.com/CAk1yvE.png)

###### 靈活運用fixture是很重要的課程！

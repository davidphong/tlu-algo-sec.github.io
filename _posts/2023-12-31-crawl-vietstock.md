---
layout: post
title:  "Hướng dẫn tự viết module crawl dữ liệu của VietStock bằng Python"
tags: [vietstock, crawl, python]
authors: [2]
---

## Mở đầu 

Trong cuộc đua không ngừng của thị trường chứng khoán, sức mạnh nằm ở sự thông tin chính xác và kịp thời. Trang web VietStock là một kho tàng dữ liệu về giá cổ phiếu, nhưng làm thế nào để khai thác hết khả năng của nó một cách hiệu quả? Trong hướng dẫn này, chúng ta sẽ cùng nhau tạo ra một công cụ Python linh hoạt, giúp tự động hóa quá trình crawl dữ liệu từ VietStock và thậm chí làm phong phú thêm với việc phân tích dữ liệu.



## Công cụ

- Trong cách làm này, module mà mình sử dụng gồm có:
  + **request** để send request tới url 
  + **datetime** để format được định dạng time thông thường 
  + **matplotlib** để vẽ biểu đồ thống kê
  + **csv** để hỗ trợ tạo, ghi và lưu file
- Tiếp đến là công cụ giúp mình bắt và đọc được requests của trang VietStock, chắc hẳn đối với anh em nó chẳng còn xa lạ mấy, đó là **BurpSuite** và extension **FoxyProxy** trên trình duyệt FireFox. 



## Giải pháp

- Đầu tiên, mình sẽ đi qua **VietStock** một chút, để xem xét mọi ngóc ngách của trang web này như thế nào. Và sau một thời gian ngồi sử dụng nó như một người bình thường thì mình cảm thấy rằng: "Nếu mình cần xem nhiều hơn mà thời gian không có  thì mình phải làm sao?". 
- Mình sử dụng tới **BurpSuite** để có thể đọc được các request của VietStock lúc này. 

```http
GET / HTTP/1.1
Host: finance.vietstock.vn
User-Agent: python-requests/2.31.0
Accept-Encoding: gzip, deflate, br
Accept: */*
Connection: close
```

- Đây là **request** có phản hồi nhưng response mà nó trả về lại là status code là **403 Forbidden**.

![Screenshot from 2023-12-19 17-24-34](https://github.com/TAS-GuarD1an/Crawl_VietStock/assets/140484031/fc69220f-0b2d-4ca1-bfa2-f91a167ca105)

- Có vẻ như hơi khó khăn trong việc kiểm tra xem tại sao lại trả về lỗi 403 nên mình đã quyết định cắm luôn **BurpSuite** vào browser để so sánh 2 request với nhau. À tới đây thì bắt đầu dùng tới FoxyProxy rùi. 
- Sau một hồi cụ thể là 10 giây thì mình lấy cái request này để so sánh :v

```http
GET / HTTP/1.1
Host: finance.vietstock.vn
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1
Te: trailers
Connection: close 
```
![Screenshot from 2023-12-19 17-23-29](https://github.com/TAS-GuarD1an/Crawl_VietStock/assets/140484031/78c50a2b-16fe-4195-9e3c-31fce14bf0b2)

- Đối với request này thì response vẫn trả về như bình thường với status code là 200. Vậy mình đã làm thế nào để phát hiện được với 1 request bình thường từ python sẽ bị trả về lỗi 403? Mình cũng khá đau đầu cho vấn đề này, nhưng sau đó thì mình đã nghĩ ra một cách, đó là mình sẽ thử xóa các header của request đi sau đó send cho server để xem response trả về như thế nào. 
  - Đầu tiên là **cookie**, response vẫn có status code là 200. Chứng tỏ cookie không ảnh hưởng gì tới request này cả. 
  - Tiếp đến là **User-Agent**, đến đây thì có vấn đề, status code đã là 403. Vậy có thể kết luận rằng request muốn được trả về dữ liệu thì cần tới User-Agent. Vậy thì cùng mình xem lại 1 chút ở request trên, tuy đã có **User-Agent: python-requests/2.31.0** nhưng sao web server vẫn trả về 403 Forbidden? Hiểu cơ bản thì ở request trên, **User-Agent** được viết từ Python HTTP Requests, nên web server đang không thể xác định được trình duyệt cụ thể. Vậy nên, nếu muốn get được dữ liệu của web này thì mình đã phải giả mạo **User-Agent** để cho web server có thể tin tưởng. 

```
Như vậy, mình đã xác định được rằng: Đối với vietstock thì mình cần phải làm gì trong requests của mình để GET được dữ liệu từ trang web. 
```



## Lập trình

### Hàm get_trading_data

- Hàm get_trading_data đóng vai trò quan trọng trong quy trình crawl dữ liệu từ trang VietStock. Hàm này lấy ra dữ liệu trading trong 30 ngày của một mã cổ phiếu bất kỳ.

![Screenshot from 2023-12-31 18-49-40](https://github.com/TAS-GuarD1an/Crawl_VietStock/assets/140484031/459e83eb-fb7a-48c0-bcca-dc59c2f4d572)

```python
def get_trading_data(stockcode):
    current_date = datetime.now()
    start_date = current_date - timedelta(days=30)
    start_date_str = start_date.strftime('%d/%m/%Y')

    url = 'https://finance.vietstock.vn/data/gettradingresult'

    headers = {
        'Host': 'finance.vietstock.vn',
        'Cookie': 'YOUR_COOKIES',  
        'User-Agent': 'Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0',
    }

    data = {
        'Code': stockcode,
        'OrderBy': '',
        'OrderDirection': 'desc',
        'PageIndex': '1',
        'PageSize': '30',
        'FromDate': start_date_str,
        'ToDate': current_date,
        'ExportType': 'default',
        'Cols': 'TKLGD,TGTGD,VHTT,TGG,DC,TGPTG,KLGDKL,GTGDKL',
        'ExchangeID': '1',
        '__RequestVerificationToken': 'WcpGXmYqtiiNjIcvN25UeW53XDTWYW6A1dS_33XTuMnj56BKNmdRvJEOPzWvKm_E-VcolXbJRaFELvsVxMEm1DyILQX_X0fivTP0dQxW62U1'
    }

    response = requests.post(url, headers=headers, data=data)
    response_json = response.json()

    return response, response_json
```

- Trước hết, hàm này xác định thời điểm bắt đầu và kết thúc của chuỗi dữ liệu cần thu thập. Bằng cách sử dụng thư viện datetime, nó tạo ra **start_date** là ngày bắt đầu, tính từ thời điểm hiện tại và lùi lại 30 ngày. Sau đó, ngày này được định dạng vào chuỗi **start_date_str** để chuẩn bị cho việc đặt tham số trong request.
- Sau khi gửi request, hàm nhận phản hồi từ web server dưới dạng JSON và trả về cả response và response_json, tạo cơ sở dữ liệu quan trọng cho việc phân tích và theo dõi. 



### Hàm save_trading_to_csv

- Trong hàm này mình sử dụng 2 tham chiếu đầu vào là **response_json** và **output_path**.
  - **response_json**: đối tượng JSON chứa thông tin giao dịch.
  - **output_path**: đường dẫn tới file csv mà dữ liệu giao dịch sẽ được lưu.

```python
def save_trading_data_to_csv(response_json, output_filepath):
    if "Data" in response_json and len(response_json["Data"]) > 0:
        with open(output_filepath, 'a', newline='') as csvfile:
            fieldnames = ['StockCode', 'Trading Date', 'Close Price']
            csv_writer = csv.DictWriter(csvfile, fieldnames=fieldnames)

            csv_writer.writeheader()
            for i in range(len(response_json["Data"])):
                stockcode = response_json["Data"][i]["StockCode"]
                close_price = str(response_json["Data"][i]["ClosePrice"])
                trading_date_str = response_json["Data"][i]["TradingDate"]

                timestamp_ms = int(trading_date_str[6:-2]) 
                trading_date_iso = datetime.utcfromtimestamp(timestamp_ms / 1000).strftime('%d/%m/%Y')

                csv_writer.writerow({'StockCode': stockcode, 'Trading Date': trading_date_iso, 'Close Price': close_price})

                print(f"{trading_date_iso}: {close_price}")

        print(f"Data saved to {output_filepath}")
    else:
        print(f"No trading data found for stock code {stockcode}")
```

- Trước hết, hàm kiểm tra xem trong **response_json** có chứa từ khóa **"Data"** hay không và liệu **"Data"** đó có dữ liệu hay không. Nếu có, chúng ta bắt đầu quá trình lưu trữ.
- Hàm mở file CSV theo đường dẫn được chỉ định trong **output_filepath**. Nếu file đã tồn tại, chúng ta chọn chế độ **'a'** để tiếp tục viết vào cuối file mà không ghi đè lên dữ liệu cũ.
- Tiếp theo, chúng ta định nghĩa các trường fieldnames trong file CSV, bao gồm **'StockCode', 'Trading Date', 'Close Price'**. Điều này giúp file csv có cái nhìn rõ ràng và bố cục hơn. 
- Sử dụng **csv.DictWriter** để ghi dữ liệu vào file CSV dựa trên các trường đã định nghĩa. Đầu tiên, ghi header bằng cách sử dụng **csv_writer.writeheader()**.
- Tiếp đến là vòng **for**, mình sử dụng vòng **for** này với mục đích duyệt tất cả các phần tử có trong **response_json["Data"]**. Và đương nhiên, khi mình đọc response thì mình có đọc được data json mà web server trả về, trong đó có các chỉ số mình cần lấy ra ví dụ như: **StockCode, ClosePrice, TradingDate**. 
- Từ đó mình có thể lấy và lưu các trường thông tin chỉ số đó vào trong file csv của mình để có thể tiện theo dõi. 
- Các câu lệnh khác, mình chỉ dùng để format lại thời gian sao cho đúng định dạng thường ngày mà chúng ta thấy, cũng như 1 số câu lệnh in ra thông báo mà thôi.



### Hàm plot_trading_data

``` python
def plot_trading_data(stockcode, date, price):
    if date and price:
        fig, ax = plt.subplots()
        ax.plot_date(date, price, '-', color='cyan')
        ax.xaxis.set_major_formatter(DateFormatter('%d/%m/%Y'))
        plt.xticks(rotation=45, color='red')
        plt.yticks(color='red')
        plt.title(f"Stock Price of {stockcode}")
        plt.xlabel("Date")
        plt.ylabel("Stock Price")
        plt.tight_layout()
        plt.show()
    else:
        print(f"No data to plot for stock code {stockcode}")
```

- Hàm này đơn giản và hiệu quả. Nó chịu trách nhiệm vẽ biểu đồ giá cổ phiếu dựa trên các thông tin về ngày và giá.
- Nếu có dữ liệu (date và price không rỗng), hàm sẽ sử dụng thư viện matplotlib để vẽ biểu đồ đường. Các trục x và y được định dạng để thể hiện ngày và giá cổ phiếu, màu sắc và kiểu đường cũng được tùy chỉnh để tạo ra một biểu đồ rõ ràng và dễ đọc.
- Nếu không có dữ liệu để vẽ, hàm sẽ thông báo rằng không có dữ liệu nào để vẽ biểu đồ cho mã cổ phiếu cụ thể đó.

![Screenshot from 2023-12-31 21-38-15](https://github.com/TAS-GuarD1an/Crawl_VietStock/assets/140484031/d99b43a4-4069-433c-9d0c-31e853f3e150)



### Main 

- Phần chương trình chính là nơi mà bạn có thể nhập mã cổ phiếu, xác định đường dẫn đến file lưu trữ, và thực hiện gọi các hàm để crawl, lưu trữ dữ liệu và vẽ biểu đồ.

```python
if __name__  == "__main__":
    #Xác định url của trang web
    url = 'https://finance.vietstock.vn/data/gettradingresult'

    #Nhập mã cổ phiếu cần tra cứu 
    stockcode_input = input("Input stock code: ")

    #Nhập đường dẫn lưu file csv
    output_file_path = "/YOUR_OUTPUT_PATH"

    #Gọi hàm để crawl dữ liệu từ VietStock
    response, response_json = get_trading_data(stockcode_input)

    #Chuẩn bị các mảng 1 chiều để lưu thông tin
    date = []
    price = []

    #Kiểm tra dữ liệu
    if "Data" in response_json and len(response_json["Data"]) > 0:
	#Lặp qua dữ liệu và lưu thông tin vào mảng 
        for i in range(len(response_json["Data"])):
            close_price = str(response_json["Data"][i]["ClosePrice"])
            trading_date_str = response_json["Data"][i]["TradingDate"]

            #Xử lý định dạng ngày giờ
            timestamp_ms = int(trading_date_str[6:-2]) 
            trading_date_iso = datetime.utcfromtimestamp(timestamp_ms / 1000).strftime('%d/%m/%Y')

            print(f"{trading_date_iso}: {close_price}")

            date.append(datetime.strptime(trading_date_iso, '%d/%m/%Y'))
            price.append(float(close_price.replace(',', '')))

        save_trading_data_to_csv(response_json, output_file_path)
        plot_trading_data(stockcode_input, date, price)
    else:
        print(f"No trading data found for stock code {stockcode_input}")
```



## Kết luận

- Sau khi viết xong tool này để mình get data từ VietStock, mình đã có thể tiết kiệm được rất nhiều thời gian cho việc theo dõi giá cổ phiếu hàng ngày chỉ với 1 câu lệnh input đơn giản. 
- Vậy thì tool này sẽ phù hợp hay có lợi với những ai? Theo cá nhân mình, tool này có thể phù hợp với tất cả mọi người, những người có ý định code để cải thiện khả năng, sử dụng tool để lấy dữ liệu về phân tích, hoặc đơn giản chỉ là theo dõi giá cổ phiếu nhưng "LƯỜI" thao tác như mình :v. 
- Ngoài ra mình cũng đã thử sử dụng tool này duới dạng 1 module, các bạn cũng có thể clone hoặc import module này để sử dụng những tính năng của nó. 

```python
from datetime import datetime, timedelta
from CrawlDone import get_trading_data, plot_trading_data, save_trading_data_to_csv

#input for stock code
stockcode_input = input("Input stock code: ")
#get trading data
respone, response_json = get_trading_data(stockcode=stockcode_input)

#Get trding data function
get_trading_data(stockcode=stockcode_input)
# print(get_trading_data(stockcode=stockcode_input))

#Save trading data to csv function 
output_file_path = "/YOUR_OUTPUT_PATH"
save_trading_data_to_csv(response_json=response_json,output_filepath=output_file_path)

#Plot trading data function
date = []
price = []
if "Data" in response_json and len(response_json["Data"]) > 0:
    for data in response_json["Data"]:
        trading_date_str = data["TradingDate"]
        close_price = str(data["ClosePrice"])
        timestamp_ms = int(trading_date_str[6:-2]) 
        trading_date_iso = datetime.utcfromtimestamp(timestamp_ms / 1000).strftime('%d/%m/%Y')
        date.append(datetime.strptime(trading_date_iso, '%d/%m/%Y'))
        price.append(float(close_price.replace(',', '')))
plot_trading_data(stockcode=stockcode_input, date=date, price=price)
```

Repo: [https://github.com/TAS-D3m0n/Crawl_VietStock](https://github.com/TAS-D3m0n/Crawl_VietStock)

**P/s: Đây là bài blog đầu tiên, cũng như kiến thức của mình vẫn còn nhiều thiếu sót. Mong được các cao nhân đọc Blog cho lời khuyên, cũng như góp ý để mình có thể học hỏi thêm.**

## Xin chân thành cảm ơn!
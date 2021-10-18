# QUIC

## Introduction

QUIC是一種基於UDP新的傳輸協定，由Google所開發。

現代使用網路習慣為行動裝置，其特點在移動時會切換無線熱點，在一般傳統TCP服務時會因切換網路了中斷連線，需再重新連線一次(Ex:JPTT等應用程式)。還有其他無線網路會遇到的問題如packet lost機率高和RTT時間長等，促使QUIC出現來解決其問題。

由QUIC發展出HTTP3標準，在現今也在許多網站上應用(如:Youtube, Facebook等)，也可以透過[HTTP/3 Check](https://http3check.net/) 工具去檢測其網站是否啟用。


在[Google QUIC文件](https://docs.google.com/document/d/1gY9-YNDNAB1eip-RTPbqphgySwSNSDHLq9D5Bty4FSU/edit)中提到QUIC與傳統TCP+TLS+HTTP2相比有以下優勢:
* Connection establishment latency
* Improved congestion control
* Multiplexing without head-of-line blocking
* Forward error correction
* Connection migration



## 實作

### 事前準備

因QUIC使用TLS1.3加密，因此在側錄封包時，也需要側錄SSLKEYLOG:
* Linux:
```
export SSLKEYLOGFILE="/home/username/sslkeylog.log" 
```
注意此設定只套用該terminal，因此需要直接在相同tty上使用chromium
```
chromium --no-sandbox
```

* Windows:
至系統內容 -> 進階 -> 環境變數新增SSLKEYLOGFILE
![](https://i.imgur.com/2JVezgA.png)


設定好後即可開始使用Wireshark側錄封包，並設定key log file:
![](https://i.imgur.com/1TQ0U9o.png)

就可以查看到所記錄到的封包內容了。


### 分析

以抓取 https://www.youtube.com 為範例(quic_lab.pcapng)，匯入sslkeylog檔案後並設定filter ip.addr == 172.217.160.77:

![](https://i.imgur.com/dYqnCOh.png)

#### QUIC Handoff

我們可以發現在一開始連線時，雙方還是使用TCP作為傳輸，到後來才使用QUIC。
這是在TCP連現階段時，Server會告訴Client自己是否支援HTTP3，如第56個封包內容:

![](https://i.imgur.com/o0CUxBR.png)


Server會發出 Header: alt-svc內含有HTTP3支援，Client端則會依據這個資訊切換成QUIC 

#### QUIC Initial & Handshake


接下來我們要更深入QUIC連線的部分，在filter加上 && quic，觀察第一個Initial封包(No.420)，並展開TLSv1.3 Client Hello:

![](https://i.imgur.com/0XSebl3.png)


可觀察到有很多資訊，我挑選幾個重點常用資訊說明:
* Extension: server_name
顧名思義就是連線主機網域名稱，範例為 www.youtube.com
* Extension: application_layer_protocol_negotiation 
簡稱ALPN，重要資訊之一，Client端會告訴Server所想使用的協定，範例為h3
* Extension: supported_version
所要使用的TLS版本，範例為TLSv1.3。另外我們其實可在Client Hello是TLS v1.2，這是以防中間網路設備會不認得1.3的資訊導致問題，但實際上在兩端點還是使用TLSv1.3
* Extension: quic_transport_parameters
QUIC是以Stream 進行傳輸，該Extension有很多傳輸設定參數:
![](https://i.imgur.com/sxWXXO8.png)


除了TLSv1.3資訊外，QUIC的標頭也有主要資訊:
* Version
使用的QUIC版本，目前大部分的網站及瀏覽器都使用v1，這邊使用draft-29
* Connection ID
這是QUIC最重要的功能之一，每個連線都會有獨立的Conection ID，能夠使端點在網路環境改變後，還能保持該QUIC連線。

> 比較有趣的是，在範例中可以看到由Client發出的SID值是0，有做了多次LAB測試(更換瀏覽器、換QUIC版本等)，也使用python [aioquic](https://github.com/aiortc/aioquic)測試，發現只要是由瀏覽器連線QUIC都會有這個狀況，若使用aioquic範例的client程式則能正常顯示SID，目前還找無相關資訊。

接著再看No.473封包，由Server回傳的Server Hello，也能在裏頭確定到Support version為TLSv1.3。

Initial結束後會再正式由Server發出Handshake(No.475~478)，會再由Client回傳ACK
若Wireshark上有設定好Profile，在Handshake 開始後就有類似TCP seq. ACK的機制了，幫助我們在後續查看Packet loss問題時可使用到。
![](https://i.imgur.com/xTEwNKy.png)


#### QUIC Stream

Handshake 結束後，準備開始Stream建立資訊，如No.490:

![](https://i.imgur.com/od5kyrq.png)

Client 準備了三條Stream可以使用，其中ID2和10僅單向(uni=1)，ID 0則可以雙向使用。
在後面封包可以觀察到Server都使用ID 0的Stream來傳送資料。



### Tshooting

#### Packet Loss 
QUIC Packet Loss狀態可以透過Wireshark上套用Packet Number來觀察，Filter設定 ip.src == 172.217.160.77 && quic

只要有發現不是連續數字，即可能為傳輸過程中有遺失封包
![](https://i.imgur.com/YgnUUsv.png)

#### Stream

QUIC是以Stream來傳送資料，若要知道整個Stream內所傳送的資料，可以使用 Follow選擇QUIC Stream:
![](https://i.imgur.com/E8jEdMl.png)





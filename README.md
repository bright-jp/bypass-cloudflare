# Cloudflare のバイパス：ベストプラクティス

[![Promo](https://github.com/luminati-io/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.jp/) 

このガイドでは、Cloudflare のセキュリティをバイパスし、ブロックされることなく Webサイトを正常にスクレイピングする方法を解説します。

- [Using Proxy Solutions](#using-proxy-solutions)
- [Spoofing HTTP Headers](#spoofing-http-headers)
- [Implementing CAPTCHA-Solving Services](#implementing-captcha-solving-services)
- [Using a Fortified Headless Browser](#using-a-fortified-headless-browser)
- [Using Cloudflare Solvers](#using-cloudflare-solvers)
- [Advanced Techniques](#advanced-techniques)
- [Incorporating Bright Data Solutions](#incorporating-bright-data-solutions)

## Cloudflare の仕組みを理解する

Cloudflare の [web application firewall](https://www.cloudflare.com/application-services/products/waf/)（WAF）は、Webアプリを DDoS やゼロデイ攻撃から保護します。グローバルネットワーク上で稼働し、リアルタイムで攻撃を阻止するとともに、独自アルゴリズムにより複数の特徴に基づいて悪意のあるボットを特定してブロックします。

- [**TLS fingerprints**](https://brightdata.jp/blog/web-data/tls-fingerprinting): JA3 fingerprints によりクライアントおよびその機能・設定を識別し、クライアントが正当かどうかを検証します。
- **[HTTP/2 fingerprints](https://www.blackhat.com/docs/eu-17/materials/eu-17-Shuster-Passive-Fingerprinting-Of-HTTP2-Clients-wp.pdf)**: HTTP/2 のパラメータを使用して、既知のボットシグネチャと照合します。
- **HTTP details**: ヘッダーや Cookie を検査し、ボットらしい構成を検出します。
- **JavaScript fingerprints**: ブラウザ、OS、ハードウェアの詳細を収集してボットを判別します。
- **Behavior analysis**: 機械学習により、リクエストレート、マウスの動き、アイドル時間を監視してボットを検出します。

ボットらしい挙動が検出されると、Cloudflare はバックグラウンドの JavaScript challenge を発行し、失敗すると CAPTCHA になります。

## Cloudflare をバイパスするテクニック

Cloudflare の独自ボット検知は完全ではないため、要件に合った最適なアプローチを見つけるには試行が必要です。

### Using Proxy Solutions

Cloudflare は、単一の IPアドレス からのリクエストが多すぎる場合にフラグを立てることでボットを検出します。これを回避するには、[premium residential proxies](https://brightdata.jp/proxy-types/residential-proxies) を使用してください。ただし user-agent のチェックが行われている場合は、user agent のスプーフィングも必要です。

### Spoofing HTTP Headers

[HTTP headers](https://brightdata.jp/blog/web-data/http-headers-for-web-scraping) はクライアントの詳細を明らかにします。Cloudflare はこれらをチェックして、多数のヘッダーを送る実ブラウザと、少数しか送らないスクレイパーを区別します。多くのスクレイパーツールでは、正規のブラウザを模倣するためにヘッダーを変更できます。一般的なヘッダーは次のとおりです。

#### User-Agent Header

`User-Agent` ヘッダーはブラウザと OS を示します。Cloudflare はボットらしい User-Agent をブロックする可能性があるため、実ブラウザ（例：Chrome、Firefox、Safari）を模倣するようにスプーフィングすると検出回避に役立ちます。たとえば、Python の [`requests` library](https://pypi.org/project/requests/) では次のように設定できます。

```python
import requests
 
headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36'
}
 
response = requests.get('http://httpbin.org/headers', headers=headers)
 
print(response.status_code)
print(response.text)
```

#### Referrer Header

Cloudflare は `Referer` ヘッダーをチェックして、リクエストの送信元を検証します。有効な URL でスプーフィングすると、そのリクエストが信頼できるものに見える可能性があります。

```python
import requests
 
headers = {
    'Referer': 'https://trusted-website.com'
}
 
response = requests.get('http://httpbin.org/headers', headers=headers)
 
print(response.status_code)
print(response.text)
```

#### Accept Headers

`Accept` ヘッダーは、クライアントが処理できるコンテンツタイプを指定します。実ブラウザの詳細な `Accept` ヘッダーを模倣すると、検出回避に役立つ場合があります。

```python
import requests
 
headers = {
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8'
}
 
response = requests.get('http://httpbin.org/headers', headers=headers)
 
print(response.status_code)
print(response.text)
```

Cloudflare はヘッダーの不整合や古いヘッダーもチェックします。たとえば、Firefox の user agent と `Sec-CH-UA-Full-Version-List` を併用すると、Firefox はこれをサポートしないためブロックされる可能性があります。

### Implementing CAPTCHA-Solving Services

Cloudflare は、他の検知手法だけでは判定が難しい場合、疑わしいクライアントに CAPTCHA を表示することがあります。同社の Turnstile システムは軽量で非対話型の challenge を実行しますが、スクレイパーに対しては対話型の CAPTCHA に切り替わることもあります。また、多くのサービスでは、これらの CAPTCHA をバイパスするために人手のソルバーを採用しています。ユースケースに最適なサービスを見つけるには、[the best CAPTCHA Solvers]( https://brightdata.jp/blog/web-data/best-captcha-solvers) に関する記事をお読みください。 

### Using a Fortified Headless Browser

Cloudflare の JavaScript challenge をバイパスするには、スクレイパーが実ブラウザの挙動を模倣する必要があります。つまり、JavaScript を実行し、Cookie を処理し、スクロール、マウス移動、クリックなどの操作をシミュレートします。[Selenium](https://www.selenium.dev/) のようなツールで実現できますが、[headless browsers](https://brightdata.jp/blog/web-data/best-headless-browsers) はしばしば正体が露呈します（例：`navigator.webdriver`）。[undetected_chromedriver](https://github.com/ultrafunkamsterdam/undetected-chromedriver) や [puppeteer-extra-plugin-stealth](https://brightdata.jp/blog/how-tos/avoid-getting-blocked-with-puppeteer-stealth) のようなプラグインは、これらの特徴を隠すのに役立ちます。

以下は undetected_chromedriver を使用する例です。

```python
import undetected_chromedriver.v2 as uc
driver = uc.Chrome()
with driver:
    driver.get('https://example.com')
```

headless browser を [a high-quality proxy service](https://brightdata.jp/proxy-types) と組み合わせることで、Cloudflare に対してさらに耐性を高められます。

```python
chrome_options = uc.ChromeOptions()

proxy_options = {
    'proxy': {
        'http': 'HTTP_PROXY_URL',
        'https': 'HTTPS_PROXY_URL'
    }
}

driver = uc.Chrome(
    options=chrome_options,
    seleniumwire_options=proxy_options
)
```

ブラウザは頻繁にアップデートされ、headless のシグネチャが露呈することがあります。また、Cloudflare の進化するアルゴリズムがそれらを悪用する可能性もあります。そのため、プラグインは定期的にメンテナンスしないと動作しなくなることがあります。

### Using Cloudflare Solvers

専用の [Cloudflare solver services](https://github.com/luminati-io/cloudflare-captcha-solver) は、基本的な保護を一時的にバイパスできる場合があります。たとえば cloudscraper は JavaScript エンジンを使用してブラウザサポートをシミュレートしますが、アップデートが古いと有効性が低下する可能性があります。

### Advanced Techniques

Cloudflare は複数のボット検知手法を採用しているため、単一のテクニックだけでは不十分です。代わりに、実ユーザーを模倣するために複数のアプローチを組み合わせてください。たとえば、強化した headless browser を使用し、人間のマウス移動（例：[B-spline curves](https://stackoverflow.com/a/48690652)）をシミュレートし、IP BAN を避けるためにレジデンシャルプロキシをローテーションし、[Hazetunnel](https://github.com/daijro/hazetunnel) のようなツールで正規のブラウザフィンガープリントを再現します。さらに CAPTCHA solver を追加すると、Cloudflare の防御をバイパスできる可能性が高まります。

## Incorporating Bright Data Solutions

[Bright Data’s Web Unlocker](https://github.com/luminati-io/web-unlocker-api) は、AI を使用してアンチボット対策（ブラウザフィンガープリント、CAPTCHA solving、IP rotation、リクエストのリトライなど）を突破し、99.99% の成功率で Cloudflare のボット検知をバイパスすることを簡素化します。最適なプロキシを自動選択し、シンプルな認証情報を提供するため、標準のプロキシサーバーと同様に使用できます。他のプロキシサーバーと同じように利用できます。

```python
import requests


host = 'brd.superproxy.io'
port = 22225

username = 'brd-customer-<customer_id>-zone-<zone_name>'
password = '<zone_password>'

proxy_url = f'http://{username}:{password}@{host}:{port}'

proxies = {
    'http': proxy_url,
    'https': proxy_url
}


url = "http://lumtest.com/myip.json"
response = requests.get(url, proxies=proxies)
print(response.json())
```

[Bright Data's Scraping Browser](https://github.com/luminati-io/scraping-browser) は、複数のプロキシを使用してサイトをアンロックするリモートブラウザ上でコードを実行することで Cloudflare をバイパスします。[Puppeteer](https://brightdata.jp/products/scraping-browser/puppeteer)、[Selenium](https://brightdata.jp/products/scraping-browser/selenium)、[Playwright](https://brightdata.jp/products/scraping-browser/playwright) と統合でき、完全な headless 体験を提供します。

## Conclusion

Cloudflare の回避は複雑になり得て、成功率もさまざまです。場当たり的にソリューションを継ぎはぎするのではなく、Web Unlocker、Scraping Browser、Web Scraper API などの [Bright Data’s offerings](https://brightdata.jp/products) の利用をご検討ください。わずか数行のコードで、複雑なソリューションの管理を心配することなく、より高い成功率を得られます。
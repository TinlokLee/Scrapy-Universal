# Scrapy-Universal
Scrapy 之通用爬虫配置
    通用爬虫实现：
        多站点爬取，将各个站点spider 共同部分保留
        不同部分提取出来单独配置，如爬取规则，页面解析方式等
        抽离处理做成一个配置文件
        再新增爬虫时，只需实现网站爬取规则和提取规则即可
   

1.  CrawlSpider 
    scrapy 提供的一个通用spider ，数据结构rule 指定爬取规则
           包括提取和页面跟进配置
           继承spider 的所有属性方法
           参数：
           link_extractor: 指定提取哪些链接，自动生成request
           allow callback follow
    Itemloader 指定提取规则，API 如下
    class scrapy.loader.ItemLoader([item, selector, response, ] **kwargs)
    实例：
    from scrapy.loader import ItemLoader
    from project.items import Product

    def parse(self, response):
        loader = ItemLoader(item=Product(), response=response)
        loader.add_xpath('name', '//div[@class="product_name"]')
        loader.add_xpath('name', '//div[@class="product_title"]')
        loader.add_xpath('price', '//p[@id="price"]')
        loader.add_css('stock', 'p#stock]')
        loader.add_value('last_updated', 'today')
        return loader.load_item()

2. 处理器 Processor
    Identity是最简单的Processor，不进行任何处理，直接返回原来的数据
    
2-1）takeFirst 返回列表第一个非空值，类似 extract_first()
              输出处理器 output processor 
    from scrapy.loader.processors import TakeFirst
    processor = TakeFirst()
    print(processor(['', 1,2,3]))

2-2）Join 相当于字符串 join() 方法，列表拼成字符串，默认使用空格分隔
    from scrapy.loader.processors import Join 
    processor = Join(',')
    print(processor(['a','x','c']))

2-3）Compose是用给定的多个函数的组合而构造的Processor
    每个输入值被传递到第一个函数，其输出再传递到第二个函数，依次类推
    直到最后一个函数返回整个处理器的输出
    from scrapy.loader.processors import Compose
    processor = Compose(str.upper, lambda s: s.strip())
    print(processor('hello world'))

2-4）MapCompose可以迭代处理一个列表输入值
    from scrapy.loader.processors import MapCompose
    processor = MapCompose(str.upper, lambda s: s.strip())
    print(processor(['Hello', 'World', 'Python']))

    >> ['HELLO', 'WORLD', 'PYTHON']

2-5）SelectJmes 可以查询JSON，传入Key，返回查询所得的Value
    安装：
        pip install jmespath
    
    from scrapy.loader.processors import SelectJmes
    proc = SelectJmes('k')
    processor = SelectJmes('k')
    print(processor({'k': 'valu'}))


3. 项目实例
    科技网站新闻列表中，所有分页的新闻详情，包括标题，正文，时间，来源等

3-1） 新建项目
    scrapy startproject caijing

    创建一个 CrawlSpider，使用crawl 模板
    scrapy genspider -t crawl pachong caijing.com 

    内容如下：
    from scrapy.linkextractors import LinkExtractor
    from scrapy.spiders import CrawlSpider, Rule

    class ChinaSpider(CrawlSpider):
        name = 'pachong'
        allowed_domains = ['caijing.com']
        # 指定起始位置
        start_urls = ['http://caijing.com/articles/']

        # 审查页面元素，找出提取规则，构造rules
        # 实现翻页查询
        rules = (
            Rule(LinkExtractor(allow=r'Items/'),
                    callback='parse_item', follow=True),
            Rule(LinkExtractor(restrict_xpaths
                    ='//div[@id="pageStyle"]//a[contains(., "下一页")]'))
        )

        def parse_item(self, response):
            data = {}
            return data 

    运行：
    scrapy crawl pachong

3-2） 解析页面
    定义Item 
    from scrapy import Field, Item 

    class MyItem(Item):
        title = Field()
        url = Field()
        ...
        source = Field()
        website = Field()

    像之前一样提取内容，就直接调用response变量的xpath()、css()等方法即可
    parse_item()方法的实现:
    def parse_item(self, response):
        item = NewsItem()
        item['title'] = response.xpath('//h1[@id="chan_newsTitle"]/text()').extract_first()
        item['url'] = response.url
        ...

        yield item
    
    再运行
    scrapy.crawl pachong


    这种提取方式非常不规整
    解决：
    用处理器Item Loader，通过add_xpath()、add_value()等方式实现配置化提取
    我们可以改写parse_item()：
    def parse_item(self, response):
        loader = PachongLoader(item=MyItem(),response=response)
        loader.add_xpath('title', '//h1...')
        loader.add_value('url', response.url)
        loader.add_xpath('text', '//div[@id="con"]...')
        ...
        yield loader.load_item()

    定义了处理器ItemLoader的子类，PachongLoader 其实现如下：
    from scrapy.loader import ItemLoader
    from scrapy.loader.processors import TakeFirst, Join, Compose
    class NewsLoader(ItemLoader):
        default_output_processor = TakeFirst()

    class PachongLoader(NewsLoader):
        text_out = Compose(Join(), lambda s: s.strip())
        source_out = Compose(Join(), lambda s: s.strip())


4. 通用配置抽取
    所有的变量都抽取，如name、allowed_domains、start_urls、rules等
    这些变量在CrawlSpider初始化的时候赋值即可
    新建一个通用的Spider来实现这个功能：
    scrapy genspider -t crawl universal universal(通用)

    在universal中，需要新建一个__init__()方法，进行初始化配置
    from scrapy.linkextractors import LinkExtractor
    from scrapy.spiders import CrawlSpider, Rule
    from scrapyuniversal.utils import get_config
    from scrapyuniversal.rules import rules

    class UniversalSpider(CrawlSpider):
        name = 'universal'
        def __init__(self, name, *args, **kwargs):
            config = get_config(name)
            self.config = config
            self.rules = self.get(config.get('rules'))
            self.start_urls = config.get('start_urls')
            self.allowed_domains = config.get('allowed_domains')

            super(UniversalSpider,self).__init__(*args, **kwargs)
        
        def parse_item(self, response):
            data = {}
            return data 




    将刚才所写的Spider内属性抽离出来配置成一个JSON，命名为pachong.json
    放到configs文件夹内，和spiders文件夹并列
    {
    "spider": "universal",
    "website": "caijing",
    "type": "新闻",
    "index": "http://caijing.com/",
    "settings": {
        "USER_AGENT":"..."
    },
    "start_urls": [
        "http://caijing.com/articles/"
    ],
    "allowed_domains": [
        "caijing.com"
    ],
    "rules": "pachong"
    }

    rules也可以单独定义成一个rules.py文件，做成配置文件
    实现Rule的分离
    from scrapy.linkextractors import LinkExtractor
    from scrapy.spiders import Rule

    rules = {
        "pachong":(
            Rule(LinkExtractor(...)),
            Rule(LinkExtractor(...))
        )
    }

    启动爬虫，需要从该配置文件中读取然后动态加载到Spider中
    需要定义一个读取该JSON文件的方法
    from os.path import realpath, dirname
    import json 

    def get_config(name):
        path = dirname(realpath(__file__)) + '/configs' + name + '.json'
        with open(path, 'r', encoding='utf-8') as f:
            return json.loads(f.read())
    定义了get_config()方法之后
    只需要向其传入JSON 配置文件的名称即可获取此JSON配置信息

    启动Spider，需要定义入口文件run.py，把它放在项目根目录下
    import sys
    from scrapy.utils.project import get_project_settings
    from scrapyuniversal.spiders.universal import UniversalSpider
    from scrapyuniversal.utils import get_config
    from scrapy.crawler import CrawlerProcess

    def run():
        name = sys.argv[1]
        custom_settings = get_config(name)
        # 爬取使用的Spider名称
        spider = custom_settings.get('spider', 'universal')
        project_settings = get_project_settings()
        settings = dict(project_settings.copy())

        # 合并配置
        settings.update(custom_settings.get('settings'))
        process = CrawlerProcess(settings)

        # 启动爬虫 
        process.crawl(spider, **{'name':name})
        process.start()

    if __name__ == '__main__':
        run()


    执行爬虫：
    python3 run.py pachong
    程序会首先读取JSON配置文件，将配置中的一些属性赋值给Spider，然后启动爬取


4-2） 解析页面配置化
    原解析函数
    def parse_item(self, response):
        loader = PachongLoader(item=MyItem(),response=response)
        loader.add_xpath('title', '//h1...')
        loader.add_value('url', response.url)
        loader.add_xpath('text', '//div[@id="con"]...')
        ...
        yield loader.load_item()

    
    这里的变量主要有Item Loader类的选用
    Item类的选用、Item Loader方法参数的定义
    我们可以在JSON文件中添加item的配置：
    "item":{
        "class": "NewItem",
        "loader": "PachongLoader",
        "attrs":{
            "title": "xpath",
            "args": [..]
        }
    },
        "url": [
            {"method": "attrs",
            "args": [
                "url"
            ]
            }
        ],
        "datetime": [
            {
                "method": "xpath",
                "args":[
                    "..."
                ],
                "re":"(...)"
            }
        ],
        ...

    将这些配置之后动态加载到parse_item()方法里
    最后，最重要的就是实现parse_item()方法
    def parse_item(self, response):
        item = self.config.get('item')
        if item:
            cls = eval(item.get('class'))()
            loader = eval(item.get('loader'))(cls, response=response)

         # 动态获取属性配置
          for key, value in item.get('attrs').items():
            for extractor in value:
                if extractor.get('method') == 'xpath':
                    loader.add_xpath(key, *extractor.get('args'), **{'re': extractor.get('re')})
                if extractor.get('method') == 'css':
                    loader.add_css(key, *extractor.get('args'), **{'re': extractor.get('re')})
                if extractor.get('method') == 'value':
                    loader.add_value(key, *extractor.get('args'), **{'re': extractor.get('re')})
                if extractor.get('method') == 'attr':
                    loader.add_value(key, getattr(response, *extractor.get('args')))
        
        yield loader.load_item()

    再次运行程序

4-3）start_url 配置
    start_urls也需要动态配置
    我们将start_urls分成两种，
    一种是直接配置URL列表，一种是调用方法生成
    它们分别定义为static和dynamic类型

    static类型配置：
    "start_urls": {
    "type": "static",
    "value": [
        "http://caijing.com/articles/"
        ]
    }

    如果start_urls是动态生成的，我们可以调用方法传参数
    "start_urls": {
    "type": "dynamic",
    "method": "pachong",
    "args": [
        5, 10 # 5-10页链接
    ]
    }

    统一新建一个urls.py文件
    def china(start, end):
        for page in range(start, end + 1):
            yield 'http://caijing.com/articles/index_' + str(page) + '.html'

    其他站点可以自行配置
    如某些链接需要用到时间戳，加密参数等，均可通过自定义方法实现

    接下来在Spider的__init__()方法中，start_urls的配置改写如下
    from scrapyuniversal import urls

    start_urls = config.get('start_urls')
    if start_urls:
        if start_urls.get('type') == 'static':
            self.start_urls = start_urls.get('value')
        elif start_urls.get('type') == 'dynamic':
            self.start_urls = list(eval('urls.' + start_urls.get('method'))(*start_urls.get('args', [])))

    至此，Spider的设置、起始链接、属性、提取方法都已经实现了全部的可配置化
    实现了Scrapy的通用爬虫，每个站点只需要修改JSON文件即可实现自由配置
    如果要更加方便的管理，可以将规则存入数据库，再对接可视化管理页面即可
    

＃selenium + python今日头条街拍美图
import json
import re
from hashlib import md5
from multiprocessing.dummy import Pool
from urllib.parse import urlencode

import pymongo
import requests
from bs4 import BeautifulSoup
from requests.exceptions import RequestException
from config import *

client = pymongo.MongoClient(MONGO_URL)
db = client[MONGO_DB]


# 获取查询的页面
def get_page(offset, keyword):
    data = {
        'aid': '24',
        'app_name': 'web_search',
        'offset': offset,
        'format': 'json',
        'keyword': keyword,
        'autoload': 'true',
        'count': '20',
        'en_qc': '1',
        'cur_tab': '1',
        'from': 'search_tab',
        'pd': 'synthesis',
        'timestamp': '1573137148204'

    }

    headers = {
        'cookie': 'tt_webid=6755810091213604366; WEATHER_CITY=%E5%8C%97%E4%BA%AC; tt_webid=6755810091213604366; '
                  'csrftoken=60cdc47f43905cdfbe54663442db1d86; s_v_web_id=b40662e1666dc76be6395af0fafdb88e; '
                  '__tasessionId=b8ms5cvr51573215031734',
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) '
                      'Chrome/78.0.3904.87 Safari/537.36'
    }
    url = "https://www.toutiao.com/api/search/content/?" + urlencode(data)
    try:
        response = requests.get(url, headers=headers)
        if response.status_code == 200:
            return response.text
        return None
    except RequestException:
        print('请求索引页出错')
        return None


# 解析查询到的页面
def parse_page(html):
    data = json.loads(html)
    if data and 'data' in data.keys():
        for item in data.get('data'):
            if item.get('cell_type') is not None:
                continue
            yield item.get('article_url')


# 获取详情页面
def get_page_detil(url):
    headers = {
        'cookie': 'tt_webid=6755810091213604366; WEATHER_CITY=%E5%8C%97%E4%BA%AC; tt_webid=6755810091213604366; '
                  'csrftoken=60cdc47f43905cdfbe54663442db1d86; s_v_web_id=b40662e1666dc76be6395af0fafdb88e; '
                  '__tasessionId=b8ms5cvr51573215031734',
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) '
                      'Chrome/78.0.3904.87 Safari/537.36'
    }
    try:
        response = requests.get(url, headers=headers)
        if response.status_code == 200:
            return response.text
        return None
    except RequestException:
        print('请求详情页出错')
        return None


# 解析详情页面
def parse_page_detil(html, url):
    soup = BeautifulSoup(html, 'lxml')
    title = soup.select('title')[0].get_text()
    pattern = re.compile('.*?gallery: JSON.parse\("(.*?)"\)', re.S)
    result = re.search(pattern, html)
    if result:
        data1 = result.group(1).replace('\\', '')
        data = json.loads(data1.replace('u002F', '/'))
        if data and 'sub_images' in data.keys():
            sub_images = data.get('sub_images')
            # 提取图片
            images = [item.get('url') for item in sub_images]
            # 下载图片
            for image in images:
                download_image(image)
            return {'title': title,
                    'url': url,
                    'image': images}


# 保存到mongodb
def save_to_mongo(result):
    if result is not None:
        if db[MONGO_TSBLE].insert(result):
            print("存储成功", result)
            return True
        return False


# 下载图片
def download_image(url):
    headers = {
        'cookie': 'tt_webid=6755810091213604366; WEATHER_CITY=%E5%8C%97%E4%BA%AC; tt_webid=6755810091213604366; '
                  'csrftoken=60cdc47f43905cdfbe54663442db1d86; s_v_web_id=b40662e1666dc76be6395af0fafdb88e; '
                  '__tasessionId=b8ms5cvr51573215031734',
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) '
                      'Chrome/78.0.3904.87 Safari/537.36'
    }
    print("正在下载:", url)
    try:
        response = requests.get(url, headers=headers)
        if response.status_code == 200:
            save_image(response.content)
        return None
    except RequestException:
        print('请求图片错误', url)
        return None


# 保存到本地文件夹
def save_image(content):
    file_path = '{0}/{1}.{2}'.format('E:/test/toutiao', md5(content).hexdigest(), '.jpg')
    with open(file_path, 'wb') as f:
        f.write(content)
        f.close()


def main(offset):
    html = get_page(offset, '街拍')
    for url in parse_page(html):
        # print(url)
        html2 = get_page_detil(url)
        if html2:
            result = parse_page_detil(html2, url)
            save_to_mongo(result)


if __name__ == '__main__':
    # for i in range(3):
    #     main(i * 20)
    pool = Pool()
    pool.map(main, [i * 20 for i in range(8)])
    pool.close()
    pool.join()

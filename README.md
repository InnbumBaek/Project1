# Project1
부동산 데이터 분석
<코드>
네이버 뉴스 크롤링 2018년~2019년까지 웹크롤링및 전처리
#크롤링시 필요한 라이브러리 불러오기
from bs4 import BeautifulSoup
import requests
import re
import datetime
from tqdm import tqdm
import sys
import time

# 크롤링할 url 생성하는 함수 만들기(검색어, 시작 날짜, 종료 날짜, 최대 페이지)
def makeUrl(search, s_date, e_date, maxpage):
    s_from = s_date.replace(".","")
    e_to = e_date.replace(".","")
    url = [f"https://search.naver.com/search.naver?where=news&query={search}&sort=0&ds={s_date}&de={e_date}&nso=so%3Ar%2Cp%3Afrom{s_from}to{e_to}%2Ca%3A&start={i}" for i in range(1, int(maxpage) * 10, 10)]
    return url  

# html에서 원하는 속성 추출하는 함수 만들기(기사, 추출하려는 속성값)
def news_attrs_crawler(articles, attrs):
    attrs_content = [i.attrs[attrs] for i in articles]
    return attrs_content

# ConnectionError방지
headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/98.0.4758.102"}

#html 생성해서 기사 크롤링하는 함수 만들기(url): 링크를 반환
def articles_crawler(url):
    #html 불러오기
    original_html = requests.get(i,headers=headers)
    html = BeautifulSoup(original_html.text, "html.parser")

    url_naver = html.select("div.group_news > ul.list_news > li div.news_area > div.news_info > div.info_group > a.info")
    url = news_attrs_crawler(url_naver,'href')
    return url


#####뉴스 크롤링 시작#####
search = input("검색어 입력: ")  
s_date = input("시작 날짜 입력(예시: 2019.01.04): ")  #2019.01.04
e_date = input("종료 날짜 입력(예시: 2019.01.05): ")   #2019.01.05
maxpage = input("최대 크롤링할 페이지 수를 입력하세요: ")

# naver url 생성
url = makeUrl(search, s_date, e_date, maxpage)

#뉴스 크롤러 실행
news_titles = []
news_url =[]
news_contents =[]
news_dates = []
news_names = []
for i in url:
    url = articles_crawler(url)
    news_url.append(url)


#제목, 링크, 내용 1차원 리스트로 꺼내는 함수 생성
def makeList(newlist, content):
    for i in content:
        for j in i:
            newlist.append(j)
    return newlist

    
#제목, 링크, 내용 담을 리스트 생성
news_url_1 = []

#1차원 리스트로 만들기(내용 제외)
makeList(news_url_1,news_url)

#NAVER 뉴스만 남기기
final_urls = []
for i in tqdm(range(len(news_url_1))):
    if "news.naver.com" in news_url_1[i]:
        final_urls.append(news_url_1[i])
    else:
        pass


# 뉴스 내용 크롤링
for i in tqdm(final_urls):
    #각 기사 html get하기
    news = requests.get(i,headers=headers)
    news_html = BeautifulSoup(news.text,"html.parser")

    time.sleep(2)

    # 언론사명 가져 오기
    names = news_html.select('#contents > div.copyright > div > p')
    if names:
      name = names[0].string[12:-38]
    elif names != names:
      time.sleep(2)
      names = news_html.select('#content > div.end_ct > div > div.copyright > div > p')
      name = names[0].string[12:-38]
    else:
      name = ''

    # 뉴스 제목 가져 오기
    title = news_html.select_one("#ct > div.media_end_head.go_trans > div.media_end_head_title > h2")
    if title == None:
        title = news_html.select_one("#content > div.end_ct > div > h2")
    
    # 뉴스 본문 가져 오기
    content = news_html.select("div#dic_area")
    if content == []:
        content = news_html.select("#articeBody")

    # 기사 텍스트만 가져 오기:  list합치기
    content = ''.join(str(content))

    # html 태그 제거 및 텍스트 다듬기
    pattern1 = '<[^>]*>' # 태그 제거
    pattern2 = r'[\n\t<>]|&lt;|&gt;' # 특수 기호 제거

    title = re.sub(pattern=pattern1, repl='', string=str(title))
    title = re.sub(pattern=pattern2, repl='', string=str(title))

    content = re.sub(pattern=pattern1, repl='', string=content)
    pattern3 = """[\n\n\n\n\n// flash 오류를 우회하기 위한 함수 추가\nfunction _flash_removeCallback() {}"""
    content = content.replace(pattern3, '')
    content = re.sub(pattern2, '', content)[1:-1] # 기사 본문 맨 앞, 맨 뒤 [ ] 제거

    news_names.append(name)
    news_titles.append(title)
    news_contents.append(content)

# 날짜 가져 오기
    try:
        html_date = news_html.select_one("div#ct> div.media_end_head.go_trans > div.media_end_head_info.nv_notrans > div.media_end_head_info_datestamp > div > span")
        news_date = html_date.attrs['data-date-time'][:10]
    except AttributeError:
        news_date = news_html.select_one("#content > div.end_ct > div > div.article_info > span > em")
        news_date = re.sub(pattern=pattern1,repl='', string=str(news_date))

    news_dates.append(news_date)


###데이터 프레임으로 만들기###
import pandas as pd

#데이터 프레임 만들기
news_df = pd.DataFrame({'date':news_dates, 'name': news_names, 'title':news_titles, 'content':news_contents, 'link': final_urls})

#중복 행 지우기
news_df = news_df.drop_duplicates(keep='first', ignore_index=True)

#데이터 프레임을 엑셀 파일로 저장
outputFileName = f'{s_date} ~ {e_date}.xlsx'
news_df.to_excel(outputFileName, sheet_name='sheet1')

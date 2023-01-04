<h1> Art Market Exploration: Data Analysis From Scratch </h1>

(***Work in Progress***)

<p align="center">

<h2>Overview</h2>

The goal of this personal project is to investigate the art market by looking into auctions. The project entails the following steps:
<ol>
  <li> Generate a dataset with info about artists (painters) by scraping 
  <a href="https://www.wikiart.org/">WikiArt.org</a> (previously done for another project) ✅</li>
  <li> Generate a dataset with data about art auctions all the artists by scraping 
  <a href="https://www.masterworks.com/">Masterworks.com</a> (previously done for another project) ✅</li>
  <li> Create an online PostgreSQL database on <a href="https://railway.app/">Railway.app</a> and upload the datasets ✅</li>
  <li> Use PostgreSQL to answer some preliminary questions ✅</li>
  <li> Investigate the structure of the art market with a time series analysis in R ✅</li>
  <li> Create an interactive Tableau dashboard displaying trends and metrics ✅</li>
  <li> Create a ML model in Python to forecast the hammer price of artwork for sale</li>
</ol>
<br />

<h2>Step 1. Artists Dataset </h2>
<a href="https://github.com/mattstell96/ArtScraper:">See this project</a"
<br />
  
<h2>Step 2. Art Auctions Dataset </h2>

Scraping MasterWork is relatively easy. For each artist’s auction sales, there is a table on the bottom page that can be easily scraped with a combination of XPATH and pandas’ read_html. However, the table is interactive and Selenium is necessary to visualize all the rows by click on the next button.

```python
## IMPORT LIBRARIES

#Libraries
from selenium import webdriver
from selenium.webdriver import Chrome
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.common.by import By
from selenium.webdriver.support import expected_conditions as EC
from webdriver_manager.chrome import ChromeDriverManager

driver = webdriver.Chrome(ChromeDriverManager().install())

import pandas as pd
from time import sleep
#ALTERNATIVE SCRAPING LIBRARIES
#from playwright.sync_api import sync_playwright
#import httpx
#import lxml
#from selectolax.parser import HTMLParser
#from dataclasses import dataclass, asdict
#from bs4 import BeautifulSoup

## CONNECTING TO URL

main_page_url = 'https://www.masterworks.com/research/artist/jean-michel-basquiat'

driver.get(main_page_url)
sleep(5)

## SCRAPE AND CRAWL
    
artists_df = pd.read_csv('/.../artists.csv').sort_values(by='name-surname')
artists_names = artists_df['name-surname'].tolist()

df = pd.DataFrame(columns=['Artist','Title','Date','Estimate','Price Realized'])
df = df[0:0]

for artist in artists_names:
    
    url = f'https://www.masterworks.com/research/artist/{artist}'
    
    df_artist = pd.DataFrame(columns=['Artist','Title','Date','Estimate','Price Realized'])
    df_artist = df_artist[0:0]

    # Get the HTML
    driver.get(url)
    sleep(5)

    while True:
        #Get the table
        try:
            table = pd.read_html(driver.find_element_by_xpath('//*[@id="root"]/div/div/div[2]/div/div[5]/div/div[1]/table').get_attribute('outerHTML'),parse_dates=True)[0]
    
            if len(table)==10:
                #Append to the dataframe
                df_artist = pd.concat([df_artist,table],ignore_index=True,sort=True)
                #Go to the next page
                
                next_page = driver.find_element(by=By.XPATH, value='//*[@id="root"]/div/div/div[2]/div/div[5]/div/div[2]/div[2]/button[2]')
                next_page.click()
                
                sleep(1)
                
            else:
                break
        except:
            print(f'{artist}:','no works scraped')
            break
        
        df_artist['Artist'] = artist
    
    df = pd.concat([df,df_artist],ignore_index=True,sort=True,join='outer')
    print(f'{artist}:',len(df_artist),'works scraped')
    
#Save file as a csv
df.to_csv(...)
```
  
<br />

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

An additional table "inflation" is necessary to compare the auctions.
```python
#In Python
import cpi

#Real_dollars = (current_dollars * new_cpi) / old_cpi

cpi.update() #Using the 'consumer Price Index for All Urban Consumer'

#Create the df

dfi = pd.DataFrame(columns=['Year','Current_CPI','2021_CPI'])

for i in range(1913,2022):
    dfi = dfi.append({'Year':i,'Current_CPI':cpi.get(i),'2021_CPI':cpi.get(2021)},ignore_index=True)

dfi.to_csv('inflation.csv',header=dfi.columns,index=False,encoding='utf-8')
file = open('inflation.csv')

dfi['Year']=dfi['Year'].astype('int64')

#Upload
conn = psycopg2.connect(dbname= "railway",user= "postgres",password="t01Do87YbBaEQuArBc0J",host= "containers-us-west-141.railway.app",port= "6602")
c = conn.cursor()
print('opened')
send_csv_to_psql(conn, 'inflation.csv', 'public.inflation')
```
  
  
<br />
  
<h2>Step 3. Create a DB Online </h2>

The db was created using Railway.app free tier subscription. To create the tables:
  
```sql
create table public.art_auctions(
title varchar(256),
artist varchar(64),
sale_date date,
lot float,
estimate varchar(64),
estimate_low float,
estimate_up float,
price_realized varchar(64),
price_realized_usd float,
primary key (title,artist,sale_date,price_realized_usd),
foreign key (artist) references public.artists_info(artist)
);

create table public.artists_info(
artist varchar(64) primary key,
birth_year int,
death_year int,
nationality varchar(64),
art_medium varchar(64),
art_movement varchar(128)
);
  
create table public.inflation(
	year int primary key,
	current_cpi float,
	cpi_2021 float
);
```
  
<br />
  
<h2>Step 4. Preliminary Answers with PostgreSQL </h2>
  
*Q1: What is the highest-selling painting each year?*<br/>
  
```sql
-- This temp view contains the real price of sales
create temp view real_sale as
select extract(year from auc.sale_date) as sale_year
	,auc.title
	,auc.artist
	,auc.price_realized_usd*inf.cpi_2021/inf.current_cpi as real_sale_price
from public.art_auctions auc
right join public.inflation inf
on extract(year from auc.sale_date) =inf.year
;

select *
from(select *
	,dense_rank() over (
						partition by sale_year
						order by real_sale_price desc
						) as sale_rank
	from real_sale) as sub
where sale_rank = 1
order by sale_year
;
```
  
A1:<br/>
<img src="https://i.imgur.com/7fzEkgw.png" alt="Answer 1"> <br/>
  
  
*Q2: What is the highest-selling artist each year?*<br/>
  
Still using the temporary table made before for Question 1.
  
```sql
select sale_year
	,artist
	,tot_sale
	,dense_rank() over (partition by sale_year
						order by tot_sale desc as sale_rank
						)
from(select sale_year, artist, sum(real_sale_price) as tot_sale
	from real_sale
	group by 1,2) as sub	 
where sale_rank = 1
order by sale_year
; 
```
A2:<br/>
<img src="https://i.imgur.com/M3KddVw.png" alt="Answer 2"> <br/>
  
  
*Q3: Show the total auction sales for each art movement every year*<br/>
  
```sql
select count(distinct art_movement)
from public.artists_info art
; -- There are 101 unique art movements in the DB

select art_movement, count(artist)
from public.artists_info art
group by 1
order by 2 desc
;-- Let's focus on the first 10:
		-- Baroque
		-- Rococo
		-- Expressionism
		-- Romanticism
		-- Baroque, Dutch Golden Age
		-- Post-Impressionism
		-- Impressionism
		-- Pop Art
		-- Realism
		-- Surrealism
      
select art_movement, count(artist)
from public.artists_info art
group by 1
order by 2 desc
;
```
A3:<br/>
<img src="https://i.imgur.com/mxnWtka.png" alt="Answer 3"> <br/>
	

<br />
  
<h2>Step 5. Time Series Analysis in R </h2>
	
The art_auctions.csv dataset has been generated by querying the raw dataset using PostgreSQL.<br/>

```r
#### PREPROCESSING ####

###Libraries
library(xts)
library(fpp2)
library(forecast)
library(vars)

###Load & Pre-Process data
art = read.csv('.../art_auctions.csv')

#Transform the df into an xts (time series object)
#We will start from 1989, to avoid missing data

art = art[art$Date_Q>=1989,]

#We can remove all the date fields now
art$Year = NULL
art$Quarter = NULL
art$Date_Q=NULL

qrtrs = seq(as.Date('1989-01-01'),length=nrow(art),by='quarters')
art_xts = xts(art,order.by=qrtrs)

#Exploratory Viz

plot(art_xts, 
     #ylim = c(0,4000),
     xlim = c(as.Date('1989-01-01'),as.Date('2021-10-01')),
     grid.col = NA,
     yaxis.right = FALSE,
     main = 'Art Auctions - Quarterly' ,
     xlab = 'Date',
     ylab = 'Total Sales',
     gpars=list(xaxt="n")
)
lines(art_xts, col = 'red', lwd=2)
axis(1, at = seq(1989, 2021, by = 2))
```
Output:<br/>
<img src="https://i.imgur.com/IfNlIVV.jpg" alt="Trend"> <br/>

```r
##1. Visualize the T-S-R components (STL Decomposition)
stl = mstl(art_xts, s.window='periodic',robust=TRUE)
autoplot(stl)

  #Findings
  #There are 2 trends
  #There is no seasonality (missing plot)
  #Unstable variance is clear in the reminder
```
Decomposition Output:<br/>
<img src="https://imgur.com/a9pESIi.png" alt="Trend"> <br/>
	  
```r
##2.Box-Cox Power Transf. To Stabilize Variance
art_xts_s = BoxCox(art_xts, lambda='auto')

plot(art_xts_s, 
     #ylim = c(0,4000),
     xlim = c(as.Date('1989-01-01'),as.Date('2021-10-01')),
     grid.col = NA,
     yaxis.right = FALSE,
     main = 'Art Auctions - Quarterly' ,
     xlab = 'Date',
     ylab = 'Total Sales',
     gpars=list(xaxt="n")
)
lines(art_xts, col = 'red', lwd=2)
axis(1, at = seq(1989, 2021, by = 2))

stl_s = mstl(art_xts_s, s.window='periodic',robust=TRUE)
autoplot(stl_s)

  #Findings
  #Box-Cox stabilized the variance (see chart and "Remainder")
  #The trend became linear
```

BoxCox & Decomposition Output:<br/>
<img src="https://imgur.com/bXz7VhW.png" alt="BoxCox"> <br/>
<img src="https://imgur.com/s25K4St.png" alt="BoxCox"> <br/>
	
```r
##3. Testing for unit root
library(fUnitRoots)

adfTest(art_xts_s, type = 'c') #The transf. series has a linear trend

  #p-value < 0.05: reject the null hypothesis of unit root


#### MODELLING ####

##ARIMA

art_arima = auto.arima(art_xts_s, approximation = TRUE, stepwise = FALSE)

art_arima
  #The time series can be modeled with a AR(3) with unit root

#Diagnostics: check residuals
checkresiduals(art_arima) #The residuals resemble white noise, the model is good
### FORECASTING ####

#1. Predict
pred_arima = forecast(art_arima, h = 4)

#2. Check performance metrics
summary(pred_arima) #Find here the error metrics

#3. Plot the prediction
plot(pred_arima,
     grid.col = NA,
     yaxis.right = FALSE,
     main = 'Art Auctions Total SAles Forecast - Quarterly' ,
     ylab = 'Total Sales',
     col='red',
     lwd=2,
)
```
ARIMA Forecast Output:<br/>
<img src="https://imgur.com/gTuTupI.png" alt="BoxCox"> <br/>
	
<br />
  
<h2>Step 6. Tableau Visualization </h2>
	
You can access the <a href="https://public.tableau.com/views/ArtVisualization/ArtDashboard?:language=en-US&publish=yes&:display_count=n&:origin=viz">dashboard on Tableau Public</a> :<br/>

<img src="https://i.imgur.com/Hb2gACT.png" alt="Forecast"> <br/>

	




  

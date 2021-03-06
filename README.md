# Mission to Mars
![alt text](https://github.com/Claude-Hanfou/Web-Scraping-Project/blob/main/Image/mars%20image.jpg "Mars")

## Objective
The goal of this project is to build a web application that scrapes various websites for data related to the Mission to Mars and displays the information in a single HTML page

The following urls were used to scrape the data 

* https://mars.nasa.gov/news/?page=0&per_page=40&order=publish_date+desc%2Ccreated_at+desc&search=&category=19%2C165%2C184%2C204&blank_scope=Latest
* https://www.jpl.nasa.gov/images/spring-sprouts-on-mars/'
* https://space-facts.com/mars/
* https://astrogeology.usgs.gov/search/results?q=hemisphere+enhanced&k1=target&v1=Mars

<img src="https://github.com/Claude-Hanfou/Web-Scraping-Project/blob/main/Image/Mission%20to%20Mars%20-%20Google%20Chrome%202021-01-31%2021-34-15_Trim.gif" width="600" height="450" /> 

### Scraping

The Jupyter Notebook file called mission_to_mars.ipynb was used to complete all of the preliminary scraping and analysis tasks, and the scrape mars python file was to convert our python script and with a function called scrape that execute all of the scraping code from jupyter notebook and return one Python dictionary containing all of the scraped data.

* The latest News Title and Paragraph Text were collected from the  NASA Mars News Site  
* The featured image was scraped from the second url,
* Mars Facts webpage was used to scrape the table containing facts about the planet including Diameter, Mass, etc.
* The USGS Astrogeology site was used to obtain high resolution images for each of Mar's hemispheres.

```python
# Dependencies
from bs4 import BeautifulSoup
import requests
import time
from splinter import Browser
import pandas as pd
from webdriver_manager.chrome import ChromeDriverManager


def init_browser():
    # @NOTE: Replace the path with your actual path to the chromedriver
    executable_path = {'executable_path': ChromeDriverManager().install()}
    return Browser('chrome', **executable_path, headless=False)


def scrape():
    browser = init_browser()
    mars_dict= {}



    #Start with the paragraph and the title_container
    url = 'https://mars.nasa.gov/news/?page=0&per_page=40&order=publish_date+desc%2Ccreated_at+desc&search=&category=19%2C165%2C184%2C204&blank_scope=Latest'
    browser.visit(url)

    html = browser.html
    soup = BeautifulSoup(html, "html.parser")

    item_list=soup.find_all("ul",class_="item_list")
    for item in item_list:
        slide=item.find_all("li",class_="slide")[0]
        mars_dict["news_p"] = slide.find('div', class_='rollover_description_inner').text.strip()  
        mars_dict["news_title"]=slide.h3.text
    


    #Featured image
    url='https://www.jpl.nasa.gov/images/spring-sprouts-on-mars/'    
    browser.visit(url)

    html = browser.html
    soup = BeautifulSoup(html, "html.parser")

    image = soup.find('div', class_='relative bg-black border border-black')
    for item in image:
        mars_dict["featured_image"]= item.find('img')['src']
    



    
    #get the table information
    url= 'https://space-facts.com/mars/'
    browser.visit(url)

    html = browser.html
    soup = BeautifulSoup(html, "html.parser")
   
    grab=pd.read_html(url)
    mars_data=pd.DataFrame(grab[0])
    mars_data.columns=['Description','Mars']
    mars_table=mars_data.set_index("Description")
    marsdata = mars_table.to_html(classes='marsdata')
    marsdata=marsdata.replace('\n', ' ')
    #store in main dictionary
    mars_dict['marsdata'] = marsdata
    


    #get the image
    url = "https://astrogeology.usgs.gov/search/results?q=hemisphere+enhanced&k1=target&v1=Mars"
    # Retrieve page with the requests module
    browser.visit(url)
    html = browser.html

    # Create BeautifulSoup object; parse with 'html.parser'
    soup = BeautifulSoup(html, 'html.parser')

    hemisphere_image_urls = []
    #Create forloop for image and title
    for i in range (4):
        time.sleep(5)
        header=browser.find_by_tag('h3')
        header[i].click()
        html = browser.html
        soup = BeautifulSoup(html, 'html.parser')
        link= soup.find('img', class_ ='wide-image')['src']
        title=soup.find('h2', class_='title').text
        image= 'https://astrogeology.usgs.gov/' + link
        dictionary={"title": title , "img_url":image}
        hemisphere_image_urls.append(dictionary)
        browser.back()
        # print(mars_dict)
        #store in main dictionary
    mars_dict['hemisphere_image_urls'] = hemisphere_image_urls

    
    # Close the browser after scraping
    browser.quit()

    #return mars
    return mars_dict
#scrape()

```

###  MongoDB and Flask Application
 MongoDB andFlask templating were used and a python app and an HTML page were created to display all of the information that was scraped from the URLs above.
 
 ```from flask import Flask, render_template, redirect
from flask_pymongo import PyMongo
import scrape_mars

app = Flask(__name__)

# Use flask_pymongo to set up mongo connection

mongo = PyMongo(app, uri="mongodb://localhost:27017/mars_app")

# Or set inline
# mongo = PyMongo(app, uri="mongodb://localhost:27017/craigslist_app")


@app.route("/")
def home():
    mars = mongo.db.collection.find_one()
    return render_template("index.html",  mars=mars)


@app.route("/scrape")
def scrape_route():
    
    mars_data = scrape_mars.scrape()
    # Update the Mongo database using update and upsert=True
    mongo.db.collection.update({}, mars_data, upsert=True)

    # Redirect back to home page
    return redirect("/",code=302)


if __name__ == "__main__":
    app.run(debug=True)
    ```

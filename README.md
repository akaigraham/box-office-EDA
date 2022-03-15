# Module 1 Final Project

## README Outline

Within this README file you will find:
1. Introduction
2. Overview of Repository Contents
3. Project Objectives
4. Overview of the Process:
    - Performance Metrics
    - Collecting Data (API Call, Web Scraping, and Provided Files)
    - Movie Runtime Analysis
    - Movie Genre Analysis
    - Gross Domestic Box Office Spend Analysis
    - Writers Analysis
5. Findings & Recommendations
6. Conclusion

## Introduction

Explore what type of films are currently doing the best at the box office, and translate those findings into actionable insights that can be used by the CEO of Microsoft.  

## Repository Contents

Within this github repository you will find the following files:
1. `README.md`
2. `student.ipynb` - jupyter notebook containing all code and analyses
3. `presentation.pdf` - non-technical presentation presenting findings, and recommendations
4. `imdb.title.ratings.csv` - CSV file containing data from IMDB, includes: IMDB Movie ID, Average Rating, and Number of Votes
5. `imdb.title.crew.csv` - CSV file containing data from IMDB, includes: IMDB Movie IDs, Director IDs, and Writer IDs
6. `imdb.name.basics.csv` - CSV file containing data on actors, writers, producers, etc. from IMDB, includes: IMDB Movie IDs, Primary Name of Individual, Birth Year of Individual, Death Year of Individual, Primary Profession of Individual, and Known For Titles (movies the individual is associated with)
7. `imdb.title.basics.csv` - CSV file containing data from IMDB, includes: IMDB Movies ID, Primary Title, Original Title, Runtime in Minutes, and Genres
8. `.png` files - png files of visualizations used in presentation

## Project Objectives

In my analysis of which movies perform best, I decided to focus on answering the following questions:

* What length movies and genres / genre combinations tend to produce the most revenue and highest ratings?
* How has domestic box office spend trended over time, and when is the best time during the year to release a movie?
* Which writers should be targeted for hire to maximize chances of writing a high grossing movie script?

## Overview of Process

## Performance Metrics

For the scope of this project, I have used revenue and average ratings as measures of movie performance and success.   

## Collecting the Data

Data used in this project comes from three sources:
1. API Calls to **[The Movie DB (TMDB)](https://www.themoviedb.org/documentation/api)**
2. Data scraped from **[Box Office Mojo](https://www.boxofficemojo.com/)**
3. IMDB CSV files provided by Flatiron

### TMDB API Calls
Pull movie data from 2000 to YTD 2020 using TMDB API.  Following API instructions provided by **[TMDB](https://www.themoviedb.org/documentation/api)**, register for API Key and use API Key to generate Access Token.  

Create headers with access token to authorize API requests.

```
headers = {'Authorization': 'Bearer {}'.format(access_token)
          ,'Content-Type': 'application/json;charset=utf-8'}
```

This API limits the number of results that can be returned in one request to 20 results per page and a maximum of 500 pages.  As a result, pulling 2000 to 2020 data in one request is not possible.  As a work around, split requests into smaller chunks so API can return data in its entirety.

Create function that returns the number of pages of results returned by API request.

```
def get_num_pages(url, headers, start_date, end_date):
    """
    Takes as input an API url, headers containing authentication information, a start date, and end date.
    Returns the number of pages of results returned by the API call as an int.
    """
    params = {'release_date.gte': start_date,
              'release_date.lte': end_date}
    returned_movies = requests.get(url=url, headers=headers, params=params).json()
    return returned_movies['total_pages']
```

Using `get_num_pages` create a function to pull in all movie data between provided start and end dates.

```
def get_movies_data(start_date, end_date, url, headers):
    """
    Takes a start date, end date, API url, and headers with authentication information.
    Uses get_num_pages function to check the number of pages returned by the API.
    Loops through all pages, requesting data from API, concatenating results to a dataframe.
    Returns dataframe of movie information between start and end date.
    """
    df = pd.DataFrame()
    num_pages = get_num_pages(url, headers, start_date, end_date)
    for i in range(1, num_pages+1):
        parameters = {'release_date.gte': start_date,
                      'release_data.lte': end_date,
                      'page': i}
        request = requests.get(url, headers=headers, params=parameters).json()
        df = pd.concat([df, pd.DataFrame(request['results'])], sort=False)

    return df
```

Create a list of start and end dates (I recommend breaking each year into quarters to ensure API limitations are not met) and use above functions to generate a dataframe of movie information.

```
df = pd.DataFrame()
url = 'https://api.themoviedb.org/3/discover/movie'

#loop through all start and end dates and make an API call for each date range
#append results to df
for i, start_date in enumerate(start_dates):
    temp_df = get_movies_data(start_date=start_date, end_date=end_dates[i], url=url, headers=headers)
    df = pd.concat([df, temp_df], sort=False)

    update_progress(i / (len(start_dates)-1))
```

Copy dataframe to `movies`, remove movies with release dates in the future and drop duplicate IDs
```
movies = df.copy()
movies = movies.loc[movies['release_date'] <= '2020-05-22']
movies.drop_duplicates(subset='id', inplace=True)
```

Pull in additional movie data.  Loop through all movie IDs in the dataframe and for each one, call Movie API (**[Documentation](https://developers.themoviedb.org/3/movies/get-movie-details)**).  Using parameters laid out in documentation did not work for me, so I resorted to using an f string to add the proper URL suffix:

```
movie_details = []
for index, movie_id in enumerate(movies['id']):
    response = requests.get(f'{url}/{movie_id}', headers=headers).json()
    movie_details.append(response)
    update_progress(index / len(movies['id']))
```

Last step is to create dataframe from `movie_details` list:

```
movies_df = pd.DataFrame(movie_details)
```

### Web Scraping Box Office Mojo
**[Box Office Mojo](https://www.boxofficemojo.com/)** presents a number of box office statistics on its website. To pull this data into a useable format, I scraped Box Office Mojo using BeautifulSoup.

```
from bs4 import BeautifulSoup
url = 'https://www.boxofficemojo.com/month/january/?grossesOption=calendarGrosses'
html_page = requests.get(url)
soup = BeautifulSoup(html_page.content, 'html.parser')
```

create list of months to scrape and empty lists to store scraped data

```
month_list = ['january', 'february', 'march', 'april', 'may',
              'june', 'july', 'august', 'september', 'october',
              'november', 'december']
months = []
years = []
gross_spend = []
```

Loop through `month_list` and extract relevant data. Given the length of the HTML class associated with the year, I created a variable for this.

```
#class used in scraping data
year_class = 'a-text-left mojo-header-column mojo-truncate mojo-field-type-year mojo-sort-column'

#loop through all months and scrape data for all years
for index, month in enumerate(month_list):
    url = f'https://www.boxofficemojo.com/month/{month}/?grossesOption=calendarGrosses'
    html_page = requests.get(url)
    soup = BeautifulSoup(html_page.content, 'html.parser')

    for td in soup.findAll('td', class_=year_class):
        years.append(td.text)
        months.append(month)

    for index, td in enumerate(soup.findAll('td', class_='a-text-right mojo-field-type-money')):
        if index%3 == 0:
            gross_spend.append(td.text[1:].replace(',', ''))
```

Last step is to create a dataframe from our scraped results

```
domestic_spend_df = pd.DataFrame([months, years, gross_spend]).transpose()
columns = ['month', 'year', 'gross_dom_spend']
domestic_spend_df.columns = columns
```

We now have a dataframe we can start analyzing

### Provided IMDB Data
Read in IMDB CSV files provided by Flatiron

```
imdb_name_basics = pd.read_csv('imdb.name.basics.csv')
imdb_title_basics = pd.read_csv('imdb.title.basics.csv')
imdb_title_crew = pd.read_csv('imdb.title.crew.csv')
imdb_title_ratings = pd.read_csv('imdb.title.ratings.csv')
```

## Movie Runtime Analysis
Use dataframe created from API calls.  Clean dataframe, handle missing values, duplicates, and NaNs. Create subsets of the dataframe for different runtime buckets. I chose to remove any movies with runtimes over 2x Standard Deviation, so we are not influenced by outliers.  I was comfortable doing this as the mean and median were very similar values.

```
movies_lt_30 = runtime_df.loc[runtime_df['runtime'] <= 30]
movies_lt_60 = runtime_df.loc[(runtime_df['runtime'] > 30) &
                              (runtime_df['runtime'] <= 60)]
movies_lt_90 = runtime_df.loc[(runtime_df['runtime'] > 60) &
                              (runtime_df['runtime'] <= 90)]
movies_lt_120 = runtime_df.loc[(runtime_df['runtime'] > 90) &
                               (runtime_df['runtime'] <= 120)]
movies_lt_150 = runtime_df.loc[(runtime_df['runtime'] > 120) &
                               (runtime_df['runtime'] <= 150)]
movies_lt_180 = runtime_df.loc[(runtime_df['runtime'] > 150) &
                               (runtime_df['runtime'] <= 180)]
movies_gt_180 = runtime_df.loc[(runtime_df['runtime'] > 180)]
```

Compare median revenue of different movie runtimes and plot results

```
x = ['<30 mins', '30-60 mins', '60-90 mins',
     '90-120 mins', '120-150 mins', '150-180 mins',
     '>180 mins']
y = [sub_30_median, sub_60_median, sub_90_median,
     sub_120_median, sub_150_median, sub_180_median,
     gt_180_median]
colors = ['lightsalmon' if (x < max(y)) else 'darkred' for x in y]

plt.figure(figsize=(10,5))
sns.barplot(x=x, y=y, palette=colors)
plt.title('Median Revenue by Runtime')
plt.xlabel('Runtime')
plt.ylabel("Median Revenue")
```

## Genre Analysis

Join provided IMDB datasets with `movies_df` created with the API calls to TMDB to pull in genre and average rating detail for our list of Movie IDs (joined DF is called `genres_df`. Clean dataframe, handle missing values, duplicates, and NaNs. Group `genres_df` by genre aggregating statistics using both `.sum()` and `.median()`

```
genres_sum_df = genres_df.groupby('genres').sum()
genres_median_df = genres_df.groupby('genres').median()
```

Sort dataframes by revenue and plot visualizations.

```
fig = plt.figure(figsize=(10, 5))
colors = ['darkred' for i in range(10)]

sns.barplot(x=genres_sum_df.index[:10],
            y='revenue',
            data=genres_sum_df[:10],
            palette=colors)

plt.xticks(rotation=90)
plt.title('Highest Grossing Genres / Genre Combinations')
plt.xlabel('Genres / Genre Combinations')
plt.ylabel('Sum of Revenue')
```

replace `data=genres_sum_df` with `data=genres_median_df` to create charts for median aggregate statistics, and update titles accordingly.  Repeat, and use the `averagerating` column in place of `revenue` to run the same analysis on ratings

## Gross Domestic Spend Analysis
Using dataframe created from Web Scrape of Box Office Mojo, group by year, summing gross domestic spend to show annual spend trends over time and create visualization.

```
year_df = domestic_spend_df.groupby('year', as_index=False).sum()
plt.figure(figsize=(10, 5))
sns.scatterplot(x='year', y='gross_dom_spend', data=year_df)
plt.title('Gross Domestic Spend Over Time')
plt.ylabel('Gross Domestic Spend')
plt.xlabel('Year')
```

Create a date column from month and year columns, convert to datetime and plot monthly gross domestic spend.

```
domestic_spend_df['date'] = domestic_spend_df['year'] + '-' + domestic_spend_df['month'] + '-1'
import datetime as dt
domestic_spend_df['date'] = domestic_spend_df['date'].apply(lambda x: dt.datetime.strptime(x, '%Y-%m-%d'))

plt.figure(figsize=(15, 5))
ax = sns.scatterplot(x='date', y='gross_dom_spend', data=domestic_spend_df)
plt.title('Gross Domestic Spend Over Time')
plt.xlabel('Year')
plt.ylabel('Gross Domestic Spend')
plt.show()
```

Group dataframe by month to evaluate seasonality of gross domestic spend throughout the year

```
plt.figure(figsize=(15, 5))
xticks = np.arange(1,13)
sns.lineplot(x='month', y='gross_dom_spend', data=domestic_spend_df)
plt.title('Gross Domestic Spend Seasonality')
xlabels = ['January', 'February', 'March', 'April',
           'May', 'June', 'July', 'August', 'September',
           'October', 'November', 'December']
plt.xticks(ticks=xticks, labels=xlabels)
plt.xlabel('Month')
plt.ylabel('Gross Domestic Spend')
plt.show()
```

Finally, produce boxplots on gross domestic spend for each month to compare distributions and variance.

```
plt.figure(figsize=(15, 5))
colors = ['lightsalmon' if (x < 5) |  (x > 7) else 'darkred' for x in range(12)]

sns.boxplot(x='month', y='gross_dom_spend', data=domestic_spend_df, palette=colors)
xticks = np.arange(12)
xlabels = ['January', 'February', 'March', 'April',
           'May', 'June', 'July', 'August', 'September',
           'October', 'November', 'December']

plt.xticks(ticks=xticks, labels=xlabels)
plt.title('Gross Domestic Spend by Month')
plt.xlabel('Month')
plt.ylabel('Gross Domestic Spend')
```

## Writers Analysis

Use provided IMDB datasets and `movies_df` created through API calls to TMDB.  
Expand crew dataframe to present each writer of a movie on their own row to create `writers` dataframe.

```
imdb_title_crew['writers_list'] = imdb_title_crew['writers'].apply(lambda x: str(x).split(','))

list_col = 'writers_list'
movie_ids = np.repeat(imdb_title_crew['tconst'].values,
                      imdb_title_crew[list_col].str.len())
writers_ids = np.concatenate(imdb_title_crew[list_col].values)
writers = pd.DataFrame(writers_ids)
```

I created a `small_movies_df` that only contains `imdb_id` and `revenue`.  Group `small_movies_df` by `imdb_id ` and join with df containing writer names and IDs.  

```
writer_details_df = imdb_name_basics.join(small_movies_df)
```

Finally, group `writer_details_df` by writer name and calculate median aggregate statistics. Create visualizations.

```
fig=plt.figure(figsize=(10, 5))
colors = ['darkred' for i in range(10)]

sns.barplot(x=grouped_writers['primary_name'][:10],
            y='revenue',
            data=grouped_writers[:10],
            palette=colors)

plt.xticks(rotation=90)
plt.title('Highest Grossing Writers')
plt.xlabel('Writers')
plt.ylabel('Median Revenue')
plt.show()
```

## Recommendations
Target summer or winter releases to coincide with spikes in gross domestic box office spend and benefit from consumer spending tailwinds

![Gross Domestic Spend Seasonlity](/data/seasonality.png)

Target runtime lengths between 2 and 3 hours

![Runtimes](/data/median_rev_by_runtime.png)

Create movies that fall into either of the following two genres:
 - Family/Fantasy/Musical to maximize potential revenue generation

![High Revenue Genres](/data/genre_median_revenue.png)


 - Comedy/Documentary/Music to maximize consumer sentiment

![High Average Rating Genres](/data/median_ratings_genre.png)

Depending on budget constraints hire one of the following 5 writers
 - Michael Crichton
 - Jim Starlin
 - Joe Robert Cole
 - Colin Trevorrow
 - Shane Morris

![Writers](/data/writers.png)

## Conclusion
Using data gathered via API calls and web scraping, I focused on what types of movies tend to produce the most revenue and highest average ratings by specifically looking at runtimes, genres, spend seasonality, and writers.  Using the insights generated within this project, the CEO of Microsoft should create a go-to-market strategy that maximizes revenue and ratings per the recommendations.  


# Italian vs. Mexican Food
---

The below script provides an analytic approach for assessing American preferences of Italian vs. Mexican food. Using data from the US Census and the Yelp API, the script randomly selects over 500 zip codes and aggregates Yelp reviews from the 20 most popular Italian and Mexican restaurants in each area. The data is then parsed and analyzed using Python Pandas and Matplotlib.


```python
# Dependencies
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import requests
import time
import json
import seaborn
from scipy.stats import ttest_ind

# Yelp API Key (Keys Hidden)
ykey_id = "XXX"
ykey_secret = "XXX"
ykey_access_token = "gl6k6JmewUhzjMVBv0I2x4Bz_NRiEggSqjlGbTaejmbzvBJXgI36FPgWoqBnEL9QQ6wU5H4h41dxPkxVjHFlawtH69m1kcXQuHev5PuWBtcdBEAbdJR0HNl3d4tpWXYx"
```

## Zip Code Sampling


```python
# Import the census data into a Pandas DataFrame
census_pd = pd.read_csv("Census_Data.csv")

# Preview the data
census_pd.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Zipcode</th>
      <th>Address</th>
      <th>Population</th>
      <th>Median Age</th>
      <th>Household Income</th>
      <th>Per Capita Income</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>15081</td>
      <td>South Heights, PA 15081, USA</td>
      <td>342</td>
      <td>50.2</td>
      <td>31500.0</td>
      <td>22177</td>
    </tr>
    <tr>
      <th>1</th>
      <td>20615</td>
      <td>Broomes Island, MD 20615, USA</td>
      <td>424</td>
      <td>43.4</td>
      <td>114375.0</td>
      <td>43920</td>
    </tr>
    <tr>
      <th>2</th>
      <td>50201</td>
      <td>Nevada, IA 50201, USA</td>
      <td>8139</td>
      <td>40.4</td>
      <td>56619.0</td>
      <td>28908</td>
    </tr>
    <tr>
      <th>3</th>
      <td>84020</td>
      <td>Draper, UT 84020, USA</td>
      <td>42751</td>
      <td>30.4</td>
      <td>89922.0</td>
      <td>33164</td>
    </tr>
    <tr>
      <th>4</th>
      <td>39097</td>
      <td>Louise, MS 39097, USA</td>
      <td>495</td>
      <td>58.0</td>
      <td>26838.0</td>
      <td>17399</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Select all zip codes with populations over 1000 from a pre-set list of 700 randomly selected zip code locations 
selected_zips = census_pd.sample(n=700)
selected_zips = selected_zips[selected_zips["Population"].astype(int) > 1000]

# Visualize
selected_zips.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Zipcode</th>
      <th>Address</th>
      <th>Population</th>
      <th>Median Age</th>
      <th>Household Income</th>
      <th>Per Capita Income</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>117</th>
      <td>76556</td>
      <td>Milano, TX 76556, USA</td>
      <td>1392</td>
      <td>50.7</td>
      <td>39375.0</td>
      <td>21201</td>
    </tr>
    <tr>
      <th>135</th>
      <td>72039</td>
      <td>Damascus, AR 72039, USA</td>
      <td>2402</td>
      <td>40.8</td>
      <td>33857.0</td>
      <td>19838</td>
    </tr>
    <tr>
      <th>389</th>
      <td>61606</td>
      <td>Peoria, IL 61606, USA</td>
      <td>7989</td>
      <td>21.9</td>
      <td>35904.0</td>
      <td>17917</td>
    </tr>
    <tr>
      <th>270</th>
      <td>47232</td>
      <td>Elizabethtown, IN 47232, USA</td>
      <td>3280</td>
      <td>37.4</td>
      <td>60128.0</td>
      <td>21838</td>
    </tr>
    <tr>
      <th>67</th>
      <td>60565</td>
      <td>Naperville, IL 60565, USA</td>
      <td>40864</td>
      <td>40.8</td>
      <td>113581.0</td>
      <td>45408</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Show the total number of zip codes that met our population cut-off
selected_zips.count()
```




    Zipcode              524
    Address              524
    Population           524
    Median Age           524
    Household Income     523
    Per Capita Income    524
    dtype: int64




```python
# Show the average population of our representive sample set
selected_zips["Population"].mean()
```




    13949.314885496184




```python
# Show the average population of our representive sample set
selected_zips["Household Income"].mean()
```




    56315.009560229446




```python
# Show the average population of our representive sample set
selected_zips["Median Age"].mean()
```




    39.937977099236626



## Yelp Data Retrieval


```python
# Create Two DataFrames to store the Italian andMexican Data 
italian_data = pd.DataFrame();
mexican_data = pd.DataFrame();

# Setup the DataFrames to have appropriate columns
italian_data["Zip Code"] = ""
italian_data["Italian Review Count"] = ""
italian_data["Italian Average Rating"] = ""
italian_data["Italian Weighted Rating"] = ""

mexican_data["Zip Code"] = ""
mexican_data["Mexican Review Count"] = ""
mexican_data["Mexican Average Rating"] = ""
mexican_data["Mexican Weighted Rating"] = ""

# Include Yelp Token
headers = {"Authorization": "Bearer gl6k6JmewUhzjMVBv0I2x4Bz_NRiEggSqjlGbTaejmbzvBJXgI36FPgWoqBnEL9QQ6wU5H4h41dxPkxVjHFlawtH69m1kcXQuHev5PuWBtcdBEAbdJR0HNl3d4tpWXYx"}
counter = 0

# Loop through every zip code
for index, row in selected_zips.iterrows():
    
    # Add to counter
    counter = counter + 1
    
    # Create two endpoint URLs:
    target_url_italian = "https://api.yelp.com/v3/businesses/search?term=Italian&location=%s" % (row["Zipcode"])
    target_url_mexican = "https://api.yelp.com/v3/businesses/search?term=Mexican&location=%s" % (row["Zipcode"])
    
    # Print the URLs to ensure logging
    print(counter)
    print(target_url_italian)
    print(target_url_mexican)
    
    # Get the Yelp Reviews
    yelp_reviews_italian = requests.get(target_url_italian, headers=headers).json()
    yelp_reviews_mexican = requests.get(target_url_mexican, headers=headers).json()
    
    # Calculate the total reviews and weighted rankings
    italian_review_count = 0
    italian_weighted_review = 0
    mexican_review_count = 0
    mexican_weighted_review = 0
    
    # Use Try-Except to handle errors
    try:
        
        # Loop through all records to calculate the review count and weighted review value
        for business in yelp_reviews_italian["businesses"]:

            italian_review_count = italian_review_count + business["review_count"]
            italian_weighted_review = italian_weighted_review + business["review_count"] * business["rating"]

        for business in yelp_reviews_mexican["businesses"]:
            mexican_review_count = mexican_review_count + business["review_count"]
            mexican_weighted_review = mexican_weighted_review + business["review_count"] * business["rating"] 
        
        # Append the data to the appropriate column of the data frames
        italian_data.set_value(index, "Zip Code", row["Zipcode"])
        italian_data.set_value(index, "Italian Review Count", italian_review_count)
        italian_data.set_value(index, "Italian Average Rating", italian_weighted_review / italian_review_count)
        italian_data.set_value(index, "Italian Weighted Rating", italian_weighted_review)

        mexican_data.set_value(index, "Zip Code", row["Zipcode"])
        mexican_data.set_value(index, "Mexican Review Count", mexican_review_count)
        mexican_data.set_value(index, "Mexican Average Rating", mexican_weighted_review / mexican_review_count)
        mexican_data.set_value(index, "Mexican Weighted Rating", mexican_weighted_review)

    except:
        print("Uh oh")
        
```

    1
    https://api.yelp.com/v3/businesses/search?term=Italian&location=76556
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76556
    2
    https://api.yelp.com/v3/businesses/search?term=Italian&location=72039
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72039
    3
    https://api.yelp.com/v3/businesses/search?term=Italian&location=61606
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=61606
    4
    https://api.yelp.com/v3/businesses/search?term=Italian&location=47232
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47232
    5
    https://api.yelp.com/v3/businesses/search?term=Italian&location=60565
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60565
    6
    https://api.yelp.com/v3/businesses/search?term=Italian&location=20634
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=20634
    7
    https://api.yelp.com/v3/businesses/search?term=Italian&location=71046
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71046
    8
    https://api.yelp.com/v3/businesses/search?term=Italian&location=76950
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76950
    9
    https://api.yelp.com/v3/businesses/search?term=Italian&location=66507
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=66507
    10
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37923
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37923
    11
    https://api.yelp.com/v3/businesses/search?term=Italian&location=30268
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30268
    12
    https://api.yelp.com/v3/businesses/search?term=Italian&location=64081
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=64081
    13
    https://api.yelp.com/v3/businesses/search?term=Italian&location=35117
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35117
    14
    https://api.yelp.com/v3/businesses/search?term=Italian&location=16930
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=16930
    15
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55735
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55735
    16
    https://api.yelp.com/v3/businesses/search?term=Italian&location=50315
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=50315
    17
    https://api.yelp.com/v3/businesses/search?term=Italian&location=62330
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=62330
    18
    https://api.yelp.com/v3/businesses/search?term=Italian&location=4357
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=4357
    19
    https://api.yelp.com/v3/businesses/search?term=Italian&location=35801
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35801
    20
    https://api.yelp.com/v3/businesses/search?term=Italian&location=63039
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=63039
    21
    https://api.yelp.com/v3/businesses/search?term=Italian&location=71353
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71353
    22
    https://api.yelp.com/v3/businesses/search?term=Italian&location=7724
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=7724
    23
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27283
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27283
    24
    https://api.yelp.com/v3/businesses/search?term=Italian&location=46236
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=46236
    25
    https://api.yelp.com/v3/businesses/search?term=Italian&location=59752
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=59752
    26
    https://api.yelp.com/v3/businesses/search?term=Italian&location=30236
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30236
    27
    https://api.yelp.com/v3/businesses/search?term=Italian&location=53576
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=53576
    28
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55802
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55802
    29
    https://api.yelp.com/v3/businesses/search?term=Italian&location=14715
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14715
    30
    https://api.yelp.com/v3/businesses/search?term=Italian&location=75783
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=75783
    31
    https://api.yelp.com/v3/businesses/search?term=Italian&location=21053
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=21053
    32
    https://api.yelp.com/v3/businesses/search?term=Italian&location=42717
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=42717
    33
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27262
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27262
    34
    https://api.yelp.com/v3/businesses/search?term=Italian&location=92543
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92543
    35
    https://api.yelp.com/v3/businesses/search?term=Italian&location=7718
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=7718
    36
    https://api.yelp.com/v3/businesses/search?term=Italian&location=21750
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=21750
    37
    https://api.yelp.com/v3/businesses/search?term=Italian&location=72833
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72833
    38
    https://api.yelp.com/v3/businesses/search?term=Italian&location=26205
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=26205
    39
    https://api.yelp.com/v3/businesses/search?term=Italian&location=63138
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=63138
    40
    https://api.yelp.com/v3/businesses/search?term=Italian&location=19057
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=19057
    41
    https://api.yelp.com/v3/businesses/search?term=Italian&location=23665
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=23665
    42
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27541
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27541
    43
    https://api.yelp.com/v3/businesses/search?term=Italian&location=97875
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97875
    44
    https://api.yelp.com/v3/businesses/search?term=Italian&location=72315
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72315
    45
    https://api.yelp.com/v3/businesses/search?term=Italian&location=54229
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54229
    46
    https://api.yelp.com/v3/businesses/search?term=Italian&location=33496
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33496
    47
    https://api.yelp.com/v3/businesses/search?term=Italian&location=23832
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=23832
    48
    https://api.yelp.com/v3/businesses/search?term=Italian&location=98101
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98101
    49
    https://api.yelp.com/v3/businesses/search?term=Italian&location=98394
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98394
    50
    https://api.yelp.com/v3/businesses/search?term=Italian&location=46323
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=46323
    51
    https://api.yelp.com/v3/businesses/search?term=Italian&location=34209
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=34209
    52
    https://api.yelp.com/v3/businesses/search?term=Italian&location=624
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=624
    53
    https://api.yelp.com/v3/businesses/search?term=Italian&location=10803
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=10803
    54
    https://api.yelp.com/v3/businesses/search?term=Italian&location=17214
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17214
    55
    https://api.yelp.com/v3/businesses/search?term=Italian&location=92285
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92285
    56
    https://api.yelp.com/v3/businesses/search?term=Italian&location=63304
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=63304
    57
    https://api.yelp.com/v3/businesses/search?term=Italian&location=38870
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=38870
    58
    https://api.yelp.com/v3/businesses/search?term=Italian&location=62449
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=62449
    59
    https://api.yelp.com/v3/businesses/search?term=Italian&location=69130
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=69130
    60
    https://api.yelp.com/v3/businesses/search?term=Italian&location=7603
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=7603
    61
    https://api.yelp.com/v3/businesses/search?term=Italian&location=52320
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=52320
    62
    https://api.yelp.com/v3/businesses/search?term=Italian&location=92392
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92392
    63
    https://api.yelp.com/v3/businesses/search?term=Italian&location=14411
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14411
    64
    https://api.yelp.com/v3/businesses/search?term=Italian&location=77039
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77039
    65
    https://api.yelp.com/v3/businesses/search?term=Italian&location=33129
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33129
    66
    https://api.yelp.com/v3/businesses/search?term=Italian&location=72007
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72007
    67
    https://api.yelp.com/v3/businesses/search?term=Italian&location=56358
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=56358
    68
    https://api.yelp.com/v3/businesses/search?term=Italian&location=56726
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=56726
    Uh oh
    69
    https://api.yelp.com/v3/businesses/search?term=Italian&location=60305
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60305
    70
    https://api.yelp.com/v3/businesses/search?term=Italian&location=75454
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=75454
    71
    https://api.yelp.com/v3/businesses/search?term=Italian&location=15656
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=15656
    72
    https://api.yelp.com/v3/businesses/search?term=Italian&location=18042
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=18042
    73
    https://api.yelp.com/v3/businesses/search?term=Italian&location=4622
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=4622
    74
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37764
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37764
    75
    https://api.yelp.com/v3/businesses/search?term=Italian&location=26807
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=26807
    76
    https://api.yelp.com/v3/businesses/search?term=Italian&location=86507
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=86507
    77
    https://api.yelp.com/v3/businesses/search?term=Italian&location=85262
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85262
    78
    https://api.yelp.com/v3/businesses/search?term=Italian&location=97366
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97366
    79
    https://api.yelp.com/v3/businesses/search?term=Italian&location=76943
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76943
    80
    https://api.yelp.com/v3/businesses/search?term=Italian&location=40060
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=40060
    81
    https://api.yelp.com/v3/businesses/search?term=Italian&location=84118
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=84118
    82
    https://api.yelp.com/v3/businesses/search?term=Italian&location=65017
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=65017
    83
    https://api.yelp.com/v3/businesses/search?term=Italian&location=77520
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77520
    84
    https://api.yelp.com/v3/businesses/search?term=Italian&location=29834
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=29834
    85
    https://api.yelp.com/v3/businesses/search?term=Italian&location=45005
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=45005
    86
    https://api.yelp.com/v3/businesses/search?term=Italian&location=32025
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=32025
    87
    https://api.yelp.com/v3/businesses/search?term=Italian&location=32187
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=32187
    88
    https://api.yelp.com/v3/businesses/search?term=Italian&location=41046
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=41046
    89
    https://api.yelp.com/v3/businesses/search?term=Italian&location=95237
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=95237
    90
    https://api.yelp.com/v3/businesses/search?term=Italian&location=57005
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57005
    91
    https://api.yelp.com/v3/businesses/search?term=Italian&location=6019
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6019
    92
    https://api.yelp.com/v3/businesses/search?term=Italian&location=36619
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=36619
    93
    https://api.yelp.com/v3/businesses/search?term=Italian&location=12831
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12831
    94
    https://api.yelp.com/v3/businesses/search?term=Italian&location=77031
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77031
    95
    https://api.yelp.com/v3/businesses/search?term=Italian&location=53588
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=53588
    96
    https://api.yelp.com/v3/businesses/search?term=Italian&location=80482
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=80482
    97
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37398
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37398
    98
    https://api.yelp.com/v3/businesses/search?term=Italian&location=48468
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48468
    99
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37345
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37345
    100
    https://api.yelp.com/v3/businesses/search?term=Italian&location=25865
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25865
    101
    https://api.yelp.com/v3/businesses/search?term=Italian&location=66614
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=66614
    102
    https://api.yelp.com/v3/businesses/search?term=Italian&location=67835
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=67835
    103
    https://api.yelp.com/v3/businesses/search?term=Italian&location=47670
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47670
    104
    https://api.yelp.com/v3/businesses/search?term=Italian&location=38049
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=38049
    105
    https://api.yelp.com/v3/businesses/search?term=Italian&location=33614
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33614
    106
    https://api.yelp.com/v3/businesses/search?term=Italian&location=10553
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=10553
    107
    https://api.yelp.com/v3/businesses/search?term=Italian&location=757
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=757
    108
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49639
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49639
    109
    https://api.yelp.com/v3/businesses/search?term=Italian&location=93545
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=93545
    110
    https://api.yelp.com/v3/businesses/search?term=Italian&location=97146
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97146
    111
    https://api.yelp.com/v3/businesses/search?term=Italian&location=87528
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=87528
    112
    https://api.yelp.com/v3/businesses/search?term=Italian&location=65243
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=65243
    113
    https://api.yelp.com/v3/businesses/search?term=Italian&location=92111
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92111
    114
    https://api.yelp.com/v3/businesses/search?term=Italian&location=730
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=730
    115
    https://api.yelp.com/v3/businesses/search?term=Italian&location=18914
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=18914
    116
    https://api.yelp.com/v3/businesses/search?term=Italian&location=94040
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=94040
    117
    https://api.yelp.com/v3/businesses/search?term=Italian&location=17980
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17980
    118
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28208
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28208
    119
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37336
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37336
    120
    https://api.yelp.com/v3/businesses/search?term=Italian&location=47932
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47932
    121
    https://api.yelp.com/v3/businesses/search?term=Italian&location=84116
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=84116
    122
    https://api.yelp.com/v3/businesses/search?term=Italian&location=48084
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48084
    123
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44143
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44143
    124
    https://api.yelp.com/v3/businesses/search?term=Italian&location=98106
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98106
    125
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49080
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49080
    126
    https://api.yelp.com/v3/businesses/search?term=Italian&location=89316
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=89316
    Uh oh
    127
    https://api.yelp.com/v3/businesses/search?term=Italian&location=52533
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=52533
    128
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49234
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49234
    129
    https://api.yelp.com/v3/businesses/search?term=Italian&location=61604
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=61604
    130
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44305
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44305
    131
    https://api.yelp.com/v3/businesses/search?term=Italian&location=35956
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35956
    132
    https://api.yelp.com/v3/businesses/search?term=Italian&location=77378
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77378
    133
    https://api.yelp.com/v3/businesses/search?term=Italian&location=25414
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25414
    134
    https://api.yelp.com/v3/businesses/search?term=Italian&location=13827
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13827
    135
    https://api.yelp.com/v3/businesses/search?term=Italian&location=3104
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=3104
    136
    https://api.yelp.com/v3/businesses/search?term=Italian&location=52141
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=52141
    137
    https://api.yelp.com/v3/businesses/search?term=Italian&location=39180
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=39180
    138
    https://api.yelp.com/v3/businesses/search?term=Italian&location=12941
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12941
    139
    https://api.yelp.com/v3/businesses/search?term=Italian&location=2740
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=2740
    140
    https://api.yelp.com/v3/businesses/search?term=Italian&location=98591
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98591
    141
    https://api.yelp.com/v3/businesses/search?term=Italian&location=12789
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12789
    142
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28395
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28395
    143
    https://api.yelp.com/v3/businesses/search?term=Italian&location=5261
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=5261
    144
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37101
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37101
    145
    https://api.yelp.com/v3/businesses/search?term=Italian&location=6062
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6062
    146
    https://api.yelp.com/v3/businesses/search?term=Italian&location=47431
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47431
    147
    https://api.yelp.com/v3/businesses/search?term=Italian&location=62340
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=62340
    148
    https://api.yelp.com/v3/businesses/search?term=Italian&location=77807
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77807
    149
    https://api.yelp.com/v3/businesses/search?term=Italian&location=60645
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60645
    150
    https://api.yelp.com/v3/businesses/search?term=Italian&location=11385
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=11385
    151
    https://api.yelp.com/v3/businesses/search?term=Italian&location=85743
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85743
    152
    https://api.yelp.com/v3/businesses/search?term=Italian&location=35005
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35005
    153
    https://api.yelp.com/v3/businesses/search?term=Italian&location=42456
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=42456
    154
    https://api.yelp.com/v3/businesses/search?term=Italian&location=61062
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=61062
    155
    https://api.yelp.com/v3/businesses/search?term=Italian&location=48221
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48221
    156
    https://api.yelp.com/v3/businesses/search?term=Italian&location=79705
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=79705
    157
    https://api.yelp.com/v3/businesses/search?term=Italian&location=79416
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=79416
    158
    https://api.yelp.com/v3/businesses/search?term=Italian&location=58647
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=58647
    159
    https://api.yelp.com/v3/businesses/search?term=Italian&location=95008
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=95008
    160
    https://api.yelp.com/v3/businesses/search?term=Italian&location=21901
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=21901
    161
    https://api.yelp.com/v3/businesses/search?term=Italian&location=24083
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=24083
    162
    https://api.yelp.com/v3/businesses/search?term=Italian&location=48416
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48416
    163
    https://api.yelp.com/v3/businesses/search?term=Italian&location=41073
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=41073
    164
    https://api.yelp.com/v3/businesses/search?term=Italian&location=38858
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=38858
    165
    https://api.yelp.com/v3/businesses/search?term=Italian&location=692
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=692
    166
    https://api.yelp.com/v3/businesses/search?term=Italian&location=58075
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=58075
    167
    https://api.yelp.com/v3/businesses/search?term=Italian&location=59820
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=59820
    168
    https://api.yelp.com/v3/businesses/search?term=Italian&location=41649
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=41649
    169
    https://api.yelp.com/v3/businesses/search?term=Italian&location=59201
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=59201
    170
    https://api.yelp.com/v3/businesses/search?term=Italian&location=57078
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57078
    171
    https://api.yelp.com/v3/businesses/search?term=Italian&location=68930
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=68930
    172
    https://api.yelp.com/v3/businesses/search?term=Italian&location=85606
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85606
    173
    https://api.yelp.com/v3/businesses/search?term=Italian&location=90302
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=90302
    174
    https://api.yelp.com/v3/businesses/search?term=Italian&location=40769
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=40769
    175
    https://api.yelp.com/v3/businesses/search?term=Italian&location=14033
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14033
    176
    https://api.yelp.com/v3/businesses/search?term=Italian&location=29113
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=29113
    177
    https://api.yelp.com/v3/businesses/search?term=Italian&location=18237
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=18237
    178
    https://api.yelp.com/v3/businesses/search?term=Italian&location=77336
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77336
    179
    https://api.yelp.com/v3/businesses/search?term=Italian&location=46617
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=46617
    180
    https://api.yelp.com/v3/businesses/search?term=Italian&location=92252
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92252
    181
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28782
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28782
    182
    https://api.yelp.com/v3/businesses/search?term=Italian&location=96783
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=96783
    183
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44276
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44276
    184
    https://api.yelp.com/v3/businesses/search?term=Italian&location=21061
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=21061
    185
    https://api.yelp.com/v3/businesses/search?term=Italian&location=54151
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54151
    186
    https://api.yelp.com/v3/businesses/search?term=Italian&location=84020
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=84020
    187
    https://api.yelp.com/v3/businesses/search?term=Italian&location=57105
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57105
    188
    https://api.yelp.com/v3/businesses/search?term=Italian&location=4107
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=4107
    189
    https://api.yelp.com/v3/businesses/search?term=Italian&location=97389
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97389
    190
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28349
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28349
    191
    https://api.yelp.com/v3/businesses/search?term=Italian&location=14304
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14304
    192
    https://api.yelp.com/v3/businesses/search?term=Italian&location=57369
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57369
    193
    https://api.yelp.com/v3/businesses/search?term=Italian&location=17018
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17018
    194
    https://api.yelp.com/v3/businesses/search?term=Italian&location=25442
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25442
    195
    https://api.yelp.com/v3/businesses/search?term=Italian&location=48386
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48386
    196
    https://api.yelp.com/v3/businesses/search?term=Italian&location=84093
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=84093
    197
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37737
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37737
    198
    https://api.yelp.com/v3/businesses/search?term=Italian&location=95490
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=95490
    199
    https://api.yelp.com/v3/businesses/search?term=Italian&location=34224
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=34224
    200
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49112
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49112
    201
    https://api.yelp.com/v3/businesses/search?term=Italian&location=46996
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=46996
    202
    https://api.yelp.com/v3/businesses/search?term=Italian&location=2382
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=2382
    203
    https://api.yelp.com/v3/businesses/search?term=Italian&location=1469
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=1469
    204
    https://api.yelp.com/v3/businesses/search?term=Italian&location=53156
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=53156
    205
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28327
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28327
    206
    https://api.yelp.com/v3/businesses/search?term=Italian&location=71834
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71834
    207
    https://api.yelp.com/v3/businesses/search?term=Italian&location=14555
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14555
    208
    https://api.yelp.com/v3/businesses/search?term=Italian&location=79703
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=79703
    209
    https://api.yelp.com/v3/businesses/search?term=Italian&location=33542
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33542
    210
    https://api.yelp.com/v3/businesses/search?term=Italian&location=43804
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=43804
    211
    https://api.yelp.com/v3/businesses/search?term=Italian&location=59802
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=59802
    212
    https://api.yelp.com/v3/businesses/search?term=Italian&location=93637
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=93637
    213
    https://api.yelp.com/v3/businesses/search?term=Italian&location=77373
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77373
    214
    https://api.yelp.com/v3/businesses/search?term=Italian&location=97022
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97022
    215
    https://api.yelp.com/v3/businesses/search?term=Italian&location=17737
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17737
    216
    https://api.yelp.com/v3/businesses/search?term=Italian&location=60137
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60137
    217
    https://api.yelp.com/v3/businesses/search?term=Italian&location=78941
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=78941
    218
    https://api.yelp.com/v3/businesses/search?term=Italian&location=97229
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97229
    219
    https://api.yelp.com/v3/businesses/search?term=Italian&location=53936
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=53936
    220
    https://api.yelp.com/v3/businesses/search?term=Italian&location=8540
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=8540
    221
    https://api.yelp.com/v3/businesses/search?term=Italian&location=93925
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=93925
    222
    https://api.yelp.com/v3/businesses/search?term=Italian&location=91405
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=91405
    223
    https://api.yelp.com/v3/businesses/search?term=Italian&location=22191
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22191
    224
    https://api.yelp.com/v3/businesses/search?term=Italian&location=22834
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22834
    225
    https://api.yelp.com/v3/businesses/search?term=Italian&location=33025
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33025
    226
    https://api.yelp.com/v3/businesses/search?term=Italian&location=14807
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14807
    227
    https://api.yelp.com/v3/businesses/search?term=Italian&location=76599
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76599
    228
    https://api.yelp.com/v3/businesses/search?term=Italian&location=38229
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=38229
    229
    https://api.yelp.com/v3/businesses/search?term=Italian&location=63128
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=63128
    230
    https://api.yelp.com/v3/businesses/search?term=Italian&location=59846
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=59846
    231
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28529
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28529
    232
    https://api.yelp.com/v3/businesses/search?term=Italian&location=31007
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=31007
    233
    https://api.yelp.com/v3/businesses/search?term=Italian&location=54979
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54979
    234
    https://api.yelp.com/v3/businesses/search?term=Italian&location=61270
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=61270
    235
    https://api.yelp.com/v3/businesses/search?term=Italian&location=50543
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=50543
    236
    https://api.yelp.com/v3/businesses/search?term=Italian&location=62613
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=62613
    237
    https://api.yelp.com/v3/businesses/search?term=Italian&location=7082
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=7082
    238
    https://api.yelp.com/v3/businesses/search?term=Italian&location=17774
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17774
    239
    https://api.yelp.com/v3/businesses/search?term=Italian&location=2056
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=2056
    240
    https://api.yelp.com/v3/businesses/search?term=Italian&location=8854
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=8854
    241
    https://api.yelp.com/v3/businesses/search?term=Italian&location=32064
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=32064
    242
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44313
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44313
    243
    https://api.yelp.com/v3/businesses/search?term=Italian&location=45780
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=45780
    244
    https://api.yelp.com/v3/businesses/search?term=Italian&location=56529
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=56529
    245
    https://api.yelp.com/v3/businesses/search?term=Italian&location=75640
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=75640
    246
    https://api.yelp.com/v3/businesses/search?term=Italian&location=13402
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13402
    247
    https://api.yelp.com/v3/businesses/search?term=Italian&location=97396
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97396
    248
    https://api.yelp.com/v3/businesses/search?term=Italian&location=54822
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54822
    249
    https://api.yelp.com/v3/businesses/search?term=Italian&location=30601
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30601
    250
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37191
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37191
    251
    https://api.yelp.com/v3/businesses/search?term=Italian&location=98358
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98358
    252
    https://api.yelp.com/v3/businesses/search?term=Italian&location=76031
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76031
    253
    https://api.yelp.com/v3/businesses/search?term=Italian&location=13637
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13637
    254
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28618
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28618
    255
    https://api.yelp.com/v3/businesses/search?term=Italian&location=32444
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=32444
    256
    https://api.yelp.com/v3/businesses/search?term=Italian&location=60622
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60622
    257
    https://api.yelp.com/v3/businesses/search?term=Italian&location=22025
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22025
    258
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27288
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27288
    259
    https://api.yelp.com/v3/businesses/search?term=Italian&location=54747
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54747
    260
    https://api.yelp.com/v3/businesses/search?term=Italian&location=53118
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=53118
    261
    https://api.yelp.com/v3/businesses/search?term=Italian&location=24137
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=24137
    262
    https://api.yelp.com/v3/businesses/search?term=Italian&location=10309
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=10309
    263
    https://api.yelp.com/v3/businesses/search?term=Italian&location=38504
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=38504
    264
    https://api.yelp.com/v3/businesses/search?term=Italian&location=29001
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=29001
    265
    https://api.yelp.com/v3/businesses/search?term=Italian&location=57315
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57315
    266
    https://api.yelp.com/v3/businesses/search?term=Italian&location=3064
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=3064
    267
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49616
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49616
    268
    https://api.yelp.com/v3/businesses/search?term=Italian&location=6516
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6516
    269
    https://api.yelp.com/v3/businesses/search?term=Italian&location=48851
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48851
    270
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27043
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27043
    271
    https://api.yelp.com/v3/businesses/search?term=Italian&location=57785
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57785
    272
    https://api.yelp.com/v3/businesses/search?term=Italian&location=19076
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=19076
    273
    https://api.yelp.com/v3/businesses/search?term=Italian&location=12010
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12010
    274
    https://api.yelp.com/v3/businesses/search?term=Italian&location=74066
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=74066
    275
    https://api.yelp.com/v3/businesses/search?term=Italian&location=70748
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=70748
    276
    https://api.yelp.com/v3/businesses/search?term=Italian&location=80831
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=80831
    277
    https://api.yelp.com/v3/businesses/search?term=Italian&location=83638
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=83638
    278
    https://api.yelp.com/v3/businesses/search?term=Italian&location=99737
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=99737
    Uh oh
    279
    https://api.yelp.com/v3/businesses/search?term=Italian&location=82932
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=82932
    280
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49874
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49874
    281
    https://api.yelp.com/v3/businesses/search?term=Italian&location=5301
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=5301
    282
    https://api.yelp.com/v3/businesses/search?term=Italian&location=3276
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=3276
    283
    https://api.yelp.com/v3/businesses/search?term=Italian&location=71031
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71031
    284
    https://api.yelp.com/v3/businesses/search?term=Italian&location=8002
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=8002
    285
    https://api.yelp.com/v3/businesses/search?term=Italian&location=33029
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33029
    286
    https://api.yelp.com/v3/businesses/search?term=Italian&location=50525
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=50525
    287
    https://api.yelp.com/v3/businesses/search?term=Italian&location=10913
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=10913
    288
    https://api.yelp.com/v3/businesses/search?term=Italian&location=90210
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=90210
    289
    https://api.yelp.com/v3/businesses/search?term=Italian&location=54733
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54733
    290
    https://api.yelp.com/v3/businesses/search?term=Italian&location=59522
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=59522
    Uh oh
    291
    https://api.yelp.com/v3/businesses/search?term=Italian&location=76135
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76135
    292
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55063
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55063
    293
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55396
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55396
    294
    https://api.yelp.com/v3/businesses/search?term=Italian&location=76525
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76525
    295
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37743
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37743
    296
    https://api.yelp.com/v3/businesses/search?term=Italian&location=23219
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=23219
    297
    https://api.yelp.com/v3/businesses/search?term=Italian&location=20143
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=20143
    298
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27549
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27549
    299
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55346
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55346
    300
    https://api.yelp.com/v3/businesses/search?term=Italian&location=33559
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33559
    301
    https://api.yelp.com/v3/businesses/search?term=Italian&location=65332
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=65332
    302
    https://api.yelp.com/v3/businesses/search?term=Italian&location=97016
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97016
    303
    https://api.yelp.com/v3/businesses/search?term=Italian&location=14806
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14806
    304
    https://api.yelp.com/v3/businesses/search?term=Italian&location=46524
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=46524
    305
    https://api.yelp.com/v3/businesses/search?term=Italian&location=4010
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=4010
    306
    https://api.yelp.com/v3/businesses/search?term=Italian&location=5156
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=5156
    307
    https://api.yelp.com/v3/businesses/search?term=Italian&location=11105
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=11105
    308
    https://api.yelp.com/v3/businesses/search?term=Italian&location=8872
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=8872
    309
    https://api.yelp.com/v3/businesses/search?term=Italian&location=17327
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17327
    310
    https://api.yelp.com/v3/businesses/search?term=Italian&location=83849
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=83849
    311
    https://api.yelp.com/v3/businesses/search?term=Italian&location=30666
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30666
    312
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44708
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44708
    313
    https://api.yelp.com/v3/businesses/search?term=Italian&location=11220
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=11220
    314
    https://api.yelp.com/v3/businesses/search?term=Italian&location=85250
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85250
    315
    https://api.yelp.com/v3/businesses/search?term=Italian&location=78611
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=78611
    316
    https://api.yelp.com/v3/businesses/search?term=Italian&location=10303
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=10303
    317
    https://api.yelp.com/v3/businesses/search?term=Italian&location=22066
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22066
    318
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55419
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55419
    319
    https://api.yelp.com/v3/businesses/search?term=Italian&location=72401
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72401
    320
    https://api.yelp.com/v3/businesses/search?term=Italian&location=22102
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22102
    321
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49781
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49781
    322
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49048
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49048
    323
    https://api.yelp.com/v3/businesses/search?term=Italian&location=18519
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=18519
    324
    https://api.yelp.com/v3/businesses/search?term=Italian&location=25427
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25427
    325
    https://api.yelp.com/v3/businesses/search?term=Italian&location=5454
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=5454
    326
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27822
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27822
    327
    https://api.yelp.com/v3/businesses/search?term=Italian&location=72434
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72434
    328
    https://api.yelp.com/v3/businesses/search?term=Italian&location=85383
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85383
    329
    https://api.yelp.com/v3/businesses/search?term=Italian&location=6824
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6824
    330
    https://api.yelp.com/v3/businesses/search?term=Italian&location=6069
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6069
    331
    https://api.yelp.com/v3/businesses/search?term=Italian&location=8859
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=8859
    Uh oh
    332
    https://api.yelp.com/v3/businesses/search?term=Italian&location=71340
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71340
    333
    https://api.yelp.com/v3/businesses/search?term=Italian&location=77434
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77434
    334
    https://api.yelp.com/v3/businesses/search?term=Italian&location=91207
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=91207
    335
    https://api.yelp.com/v3/businesses/search?term=Italian&location=12723
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12723
    336
    https://api.yelp.com/v3/businesses/search?term=Italian&location=42366
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=42366
    337
    https://api.yelp.com/v3/businesses/search?term=Italian&location=62025
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=62025
    338
    https://api.yelp.com/v3/businesses/search?term=Italian&location=35951
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35951
    339
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37727
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37727
    340
    https://api.yelp.com/v3/businesses/search?term=Italian&location=72722
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72722
    341
    https://api.yelp.com/v3/businesses/search?term=Italian&location=73014
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=73014
    342
    https://api.yelp.com/v3/businesses/search?term=Italian&location=75457
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=75457
    343
    https://api.yelp.com/v3/businesses/search?term=Italian&location=97633
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97633
    344
    https://api.yelp.com/v3/businesses/search?term=Italian&location=20886
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=20886
    345
    https://api.yelp.com/v3/businesses/search?term=Italian&location=2169
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=2169
    346
    https://api.yelp.com/v3/businesses/search?term=Italian&location=29170
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=29170
    347
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27317
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27317
    348
    https://api.yelp.com/v3/businesses/search?term=Italian&location=98362
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98362
    349
    https://api.yelp.com/v3/businesses/search?term=Italian&location=15206
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=15206
    350
    https://api.yelp.com/v3/businesses/search?term=Italian&location=43758
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=43758
    351
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28732
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28732
    352
    https://api.yelp.com/v3/businesses/search?term=Italian&location=77880
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77880
    353
    https://api.yelp.com/v3/businesses/search?term=Italian&location=30564
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30564
    354
    https://api.yelp.com/v3/businesses/search?term=Italian&location=23228
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=23228
    355
    https://api.yelp.com/v3/businesses/search?term=Italian&location=25253
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25253
    356
    https://api.yelp.com/v3/businesses/search?term=Italian&location=17051
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17051
    357
    https://api.yelp.com/v3/businesses/search?term=Italian&location=23146
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=23146
    358
    https://api.yelp.com/v3/businesses/search?term=Italian&location=32535
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=32535
    359
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49893
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49893
    360
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49036
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49036
    361
    https://api.yelp.com/v3/businesses/search?term=Italian&location=16153
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=16153
    362
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49098
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49098
    363
    https://api.yelp.com/v3/businesses/search?term=Italian&location=98012
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98012
    364
    https://api.yelp.com/v3/businesses/search?term=Italian&location=36558
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=36558
    365
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28076
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28076
    366
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55342
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55342
    367
    https://api.yelp.com/v3/businesses/search?term=Italian&location=60421
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60421
    368
    https://api.yelp.com/v3/businesses/search?term=Italian&location=79764
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=79764
    369
    https://api.yelp.com/v3/businesses/search?term=Italian&location=1430
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=1430
    370
    https://api.yelp.com/v3/businesses/search?term=Italian&location=5658
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=5658
    371
    https://api.yelp.com/v3/businesses/search?term=Italian&location=45036
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=45036
    372
    https://api.yelp.com/v3/businesses/search?term=Italian&location=45684
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=45684
    373
    https://api.yelp.com/v3/businesses/search?term=Italian&location=6377
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6377
    374
    https://api.yelp.com/v3/businesses/search?term=Italian&location=92843
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92843
    375
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49411
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49411
    376
    https://api.yelp.com/v3/businesses/search?term=Italian&location=77328
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77328
    377
    https://api.yelp.com/v3/businesses/search?term=Italian&location=92310
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92310
    378
    https://api.yelp.com/v3/businesses/search?term=Italian&location=60018
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60018
    379
    https://api.yelp.com/v3/businesses/search?term=Italian&location=67661
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=67661
    380
    https://api.yelp.com/v3/businesses/search?term=Italian&location=78214
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=78214
    381
    https://api.yelp.com/v3/businesses/search?term=Italian&location=19372
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=19372
    382
    https://api.yelp.com/v3/businesses/search?term=Italian&location=13326
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13326
    383
    https://api.yelp.com/v3/businesses/search?term=Italian&location=31771
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=31771
    384
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44147
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44147
    385
    https://api.yelp.com/v3/businesses/search?term=Italian&location=12759
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12759
    386
    https://api.yelp.com/v3/businesses/search?term=Italian&location=99506
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=99506
    387
    https://api.yelp.com/v3/businesses/search?term=Italian&location=4268
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=4268
    388
    https://api.yelp.com/v3/businesses/search?term=Italian&location=90003
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=90003
    389
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49269
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49269
    390
    https://api.yelp.com/v3/businesses/search?term=Italian&location=68005
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=68005
    391
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44629
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44629
    392
    https://api.yelp.com/v3/businesses/search?term=Italian&location=35907
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35907
    393
    https://api.yelp.com/v3/businesses/search?term=Italian&location=93428
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=93428
    394
    https://api.yelp.com/v3/businesses/search?term=Italian&location=66052
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=66052
    395
    https://api.yelp.com/v3/businesses/search?term=Italian&location=32409
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=32409
    396
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55069
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55069
    397
    https://api.yelp.com/v3/businesses/search?term=Italian&location=48191
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48191
    398
    https://api.yelp.com/v3/businesses/search?term=Italian&location=42025
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=42025
    399
    https://api.yelp.com/v3/businesses/search?term=Italian&location=25401
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25401
    400
    https://api.yelp.com/v3/businesses/search?term=Italian&location=4418
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=4418
    401
    https://api.yelp.com/v3/businesses/search?term=Italian&location=85353
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85353
    402
    https://api.yelp.com/v3/businesses/search?term=Italian&location=12148
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12148
    403
    https://api.yelp.com/v3/businesses/search?term=Italian&location=8234
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=8234
    404
    https://api.yelp.com/v3/businesses/search?term=Italian&location=11693
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=11693
    405
    https://api.yelp.com/v3/businesses/search?term=Italian&location=63935
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=63935
    406
    https://api.yelp.com/v3/businesses/search?term=Italian&location=76638
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76638
    407
    https://api.yelp.com/v3/businesses/search?term=Italian&location=13335
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13335
    408
    https://api.yelp.com/v3/businesses/search?term=Italian&location=83313
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=83313
    409
    https://api.yelp.com/v3/businesses/search?term=Italian&location=69334
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=69334
    410
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27298
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27298
    411
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28635
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28635
    412
    https://api.yelp.com/v3/businesses/search?term=Italian&location=1922
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=1922
    413
    https://api.yelp.com/v3/businesses/search?term=Italian&location=98270
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98270
    414
    https://api.yelp.com/v3/businesses/search?term=Italian&location=34744
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=34744
    415
    https://api.yelp.com/v3/businesses/search?term=Italian&location=23602
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=23602
    416
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55976
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55976
    417
    https://api.yelp.com/v3/businesses/search?term=Italian&location=1585
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=1585
    418
    https://api.yelp.com/v3/businesses/search?term=Italian&location=30152
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30152
    419
    https://api.yelp.com/v3/businesses/search?term=Italian&location=69101
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=69101
    420
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27249
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27249
    421
    https://api.yelp.com/v3/businesses/search?term=Italian&location=94556
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=94556
    422
    https://api.yelp.com/v3/businesses/search?term=Italian&location=13495
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13495
    423
    https://api.yelp.com/v3/businesses/search?term=Italian&location=43050
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=43050
    424
    https://api.yelp.com/v3/businesses/search?term=Italian&location=71405
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71405
    425
    https://api.yelp.com/v3/businesses/search?term=Italian&location=30177
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30177
    426
    https://api.yelp.com/v3/businesses/search?term=Italian&location=22642
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22642
    427
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37067
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37067
    428
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27358
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27358
    429
    https://api.yelp.com/v3/businesses/search?term=Italian&location=75149
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=75149
    430
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37080
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37080
    431
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28436
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28436
    432
    https://api.yelp.com/v3/businesses/search?term=Italian&location=1527
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=1527
    433
    https://api.yelp.com/v3/businesses/search?term=Italian&location=95240
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=95240
    434
    https://api.yelp.com/v3/businesses/search?term=Italian&location=53018
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=53018
    435
    https://api.yelp.com/v3/businesses/search?term=Italian&location=83634
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=83634
    436
    https://api.yelp.com/v3/businesses/search?term=Italian&location=10984
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=10984
    437
    https://api.yelp.com/v3/businesses/search?term=Italian&location=69347
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=69347
    438
    https://api.yelp.com/v3/businesses/search?term=Italian&location=6801
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6801
    439
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28454
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28454
    440
    https://api.yelp.com/v3/businesses/search?term=Italian&location=6883
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6883
    441
    https://api.yelp.com/v3/businesses/search?term=Italian&location=79118
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=79118
    442
    https://api.yelp.com/v3/businesses/search?term=Italian&location=3077
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=3077
    443
    https://api.yelp.com/v3/businesses/search?term=Italian&location=78004
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=78004
    444
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55435
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55435
    445
    https://api.yelp.com/v3/businesses/search?term=Italian&location=17938
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17938
    446
    https://api.yelp.com/v3/businesses/search?term=Italian&location=60091
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60091
    447
    https://api.yelp.com/v3/businesses/search?term=Italian&location=2717
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=2717
    448
    https://api.yelp.com/v3/businesses/search?term=Italian&location=16870
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=16870
    449
    https://api.yelp.com/v3/businesses/search?term=Italian&location=6422
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6422
    450
    https://api.yelp.com/v3/businesses/search?term=Italian&location=98409
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98409
    451
    https://api.yelp.com/v3/businesses/search?term=Italian&location=98051
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98051
    452
    https://api.yelp.com/v3/businesses/search?term=Italian&location=85233
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85233
    453
    https://api.yelp.com/v3/businesses/search?term=Italian&location=14131
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14131
    454
    https://api.yelp.com/v3/businesses/search?term=Italian&location=66088
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=66088
    455
    https://api.yelp.com/v3/businesses/search?term=Italian&location=47003
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47003
    456
    https://api.yelp.com/v3/businesses/search?term=Italian&location=46312
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=46312
    457
    https://api.yelp.com/v3/businesses/search?term=Italian&location=24281
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=24281
    458
    https://api.yelp.com/v3/businesses/search?term=Italian&location=47591
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47591
    459
    https://api.yelp.com/v3/businesses/search?term=Italian&location=96818
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=96818
    460
    https://api.yelp.com/v3/businesses/search?term=Italian&location=41564
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=41564
    461
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44867
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44867
    462
    https://api.yelp.com/v3/businesses/search?term=Italian&location=2145
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=2145
    463
    https://api.yelp.com/v3/businesses/search?term=Italian&location=13068
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13068
    464
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44273
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44273
    465
    https://api.yelp.com/v3/businesses/search?term=Italian&location=68147
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=68147
    466
    https://api.yelp.com/v3/businesses/search?term=Italian&location=957
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=957
    467
    https://api.yelp.com/v3/businesses/search?term=Italian&location=38224
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=38224
    468
    https://api.yelp.com/v3/businesses/search?term=Italian&location=19023
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=19023
    469
    https://api.yelp.com/v3/businesses/search?term=Italian&location=31405
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=31405
    470
    https://api.yelp.com/v3/businesses/search?term=Italian&location=47952
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47952
    471
    https://api.yelp.com/v3/businesses/search?term=Italian&location=58503
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=58503
    472
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28371
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28371
    473
    https://api.yelp.com/v3/businesses/search?term=Italian&location=76310
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76310
    474
    https://api.yelp.com/v3/businesses/search?term=Italian&location=48659
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48659
    475
    https://api.yelp.com/v3/businesses/search?term=Italian&location=25559
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25559
    476
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55433
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55433
    477
    https://api.yelp.com/v3/businesses/search?term=Italian&location=29468
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=29468
    478
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28697
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28697
    479
    https://api.yelp.com/v3/businesses/search?term=Italian&location=63038
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=63038
    480
    https://api.yelp.com/v3/businesses/search?term=Italian&location=99661
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=99661
    Uh oh
    481
    https://api.yelp.com/v3/businesses/search?term=Italian&location=74956
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=74956
    482
    https://api.yelp.com/v3/businesses/search?term=Italian&location=10007
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=10007
    483
    https://api.yelp.com/v3/businesses/search?term=Italian&location=92115
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92115
    484
    https://api.yelp.com/v3/businesses/search?term=Italian&location=67839
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=67839
    485
    https://api.yelp.com/v3/businesses/search?term=Italian&location=7606
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=7606
    486
    https://api.yelp.com/v3/businesses/search?term=Italian&location=85553
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85553
    487
    https://api.yelp.com/v3/businesses/search?term=Italian&location=22952
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22952
    488
    https://api.yelp.com/v3/businesses/search?term=Italian&location=57004
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57004
    489
    https://api.yelp.com/v3/businesses/search?term=Italian&location=54022
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54022
    490
    https://api.yelp.com/v3/businesses/search?term=Italian&location=68317
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=68317
    491
    https://api.yelp.com/v3/businesses/search?term=Italian&location=95246
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=95246
    492
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44093
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44093
    493
    https://api.yelp.com/v3/businesses/search?term=Italian&location=12804
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12804
    494
    https://api.yelp.com/v3/businesses/search?term=Italian&location=88337
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=88337
    495
    https://api.yelp.com/v3/businesses/search?term=Italian&location=76706
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76706
    496
    https://api.yelp.com/v3/businesses/search?term=Italian&location=11722
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=11722
    497
    https://api.yelp.com/v3/businesses/search?term=Italian&location=60433
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60433
    498
    https://api.yelp.com/v3/businesses/search?term=Italian&location=71957
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71957
    499
    https://api.yelp.com/v3/businesses/search?term=Italian&location=36701
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=36701
    500
    https://api.yelp.com/v3/businesses/search?term=Italian&location=58369
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=58369
    501
    https://api.yelp.com/v3/businesses/search?term=Italian&location=30309
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30309
    502
    https://api.yelp.com/v3/businesses/search?term=Italian&location=30120
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30120
    503
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49058
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49058
    504
    https://api.yelp.com/v3/businesses/search?term=Italian&location=47393
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47393
    505
    https://api.yelp.com/v3/businesses/search?term=Italian&location=50002
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=50002
    506
    https://api.yelp.com/v3/businesses/search?term=Italian&location=48469
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48469
    507
    https://api.yelp.com/v3/businesses/search?term=Italian&location=35475
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35475
    508
    https://api.yelp.com/v3/businesses/search?term=Italian&location=94539
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=94539
    509
    https://api.yelp.com/v3/businesses/search?term=Italian&location=3048
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=3048
    510
    https://api.yelp.com/v3/businesses/search?term=Italian&location=36736
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=36736
    511
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49415
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49415
    512
    https://api.yelp.com/v3/businesses/search?term=Italian&location=25571
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25571
    513
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55060
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55060
    514
    https://api.yelp.com/v3/businesses/search?term=Italian&location=62063
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=62063
    515
    https://api.yelp.com/v3/businesses/search?term=Italian&location=22027
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22027
    516
    https://api.yelp.com/v3/businesses/search?term=Italian&location=67530
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=67530
    517
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44510
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44510
    518
    https://api.yelp.com/v3/businesses/search?term=Italian&location=95051
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=95051
    519
    https://api.yelp.com/v3/businesses/search?term=Italian&location=97321
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97321
    520
    https://api.yelp.com/v3/businesses/search?term=Italian&location=5483
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=5483
    521
    https://api.yelp.com/v3/businesses/search?term=Italian&location=60468
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60468
    522
    https://api.yelp.com/v3/businesses/search?term=Italian&location=50201
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=50201
    523
    https://api.yelp.com/v3/businesses/search?term=Italian&location=73951
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=73951
    524
    https://api.yelp.com/v3/businesses/search?term=Italian&location=42031
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=42031
    


```python
# Preview Italian Data
italian_data.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Zip Code</th>
      <th>Italian Review Count</th>
      <th>Italian Average Rating</th>
      <th>Italian Weighted Rating</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>117</th>
      <td>76556</td>
      <td>63</td>
      <td>3.78571</td>
      <td>238.5</td>
    </tr>
    <tr>
      <th>135</th>
      <td>72039</td>
      <td>266</td>
      <td>3.81955</td>
      <td>1016</td>
    </tr>
    <tr>
      <th>389</th>
      <td>61606</td>
      <td>66</td>
      <td>3.2197</td>
      <td>212.5</td>
    </tr>
    <tr>
      <th>270</th>
      <td>47232</td>
      <td>420</td>
      <td>3.77857</td>
      <td>1587</td>
    </tr>
    <tr>
      <th>67</th>
      <td>60565</td>
      <td>2829</td>
      <td>3.92824</td>
      <td>11113</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Preview Mexican Data
mexican_data.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Zip Code</th>
      <th>Mexican Review Count</th>
      <th>Mexican Average Rating</th>
      <th>Mexican Weighted Rating</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>117</th>
      <td>76556</td>
      <td>97</td>
      <td>4.1134</td>
      <td>399</td>
    </tr>
    <tr>
      <th>135</th>
      <td>72039</td>
      <td>256</td>
      <td>4.11133</td>
      <td>1052.5</td>
    </tr>
    <tr>
      <th>389</th>
      <td>61606</td>
      <td>378</td>
      <td>3.64286</td>
      <td>1377</td>
    </tr>
    <tr>
      <th>270</th>
      <td>47232</td>
      <td>222</td>
      <td>4.16892</td>
      <td>925.5</td>
    </tr>
    <tr>
      <th>67</th>
      <td>60565</td>
      <td>2842</td>
      <td>3.94053</td>
      <td>11199</td>
    </tr>
  </tbody>
</table>
</div>



## Calculate Summaries


```python
# Total Mexican Reviews
mexican_data["Mexican Review Count"].sum()
```




    476889




```python
# Total Italian Reviews
italian_data["Italian Review Count"].sum()
```




    573733




```python
# Average Mexican Rating
mexican_data["Mexican Weighted Rating"].sum() / mexican_data["Mexican Review Count"].sum()
```




    3.909732663156416




```python
# Average Italian Rating
italian_data["Italian Weighted Rating"].sum() / italian_data["Italian Review Count"].sum()
```




    3.9446641556263975




```python
# Combine DataFrames into a single DataFrame
combined_data = pd.merge(mexican_data, italian_data, on="Zip Code")
combined_data.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Zip Code</th>
      <th>Mexican Review Count</th>
      <th>Mexican Average Rating</th>
      <th>Mexican Weighted Rating</th>
      <th>Italian Review Count</th>
      <th>Italian Average Rating</th>
      <th>Italian Weighted Rating</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>76556</td>
      <td>97</td>
      <td>4.1134</td>
      <td>399</td>
      <td>63</td>
      <td>3.78571</td>
      <td>238.5</td>
    </tr>
    <tr>
      <th>1</th>
      <td>72039</td>
      <td>256</td>
      <td>4.11133</td>
      <td>1052.5</td>
      <td>266</td>
      <td>3.81955</td>
      <td>1016</td>
    </tr>
    <tr>
      <th>2</th>
      <td>61606</td>
      <td>378</td>
      <td>3.64286</td>
      <td>1377</td>
      <td>66</td>
      <td>3.2197</td>
      <td>212.5</td>
    </tr>
    <tr>
      <th>3</th>
      <td>47232</td>
      <td>222</td>
      <td>4.16892</td>
      <td>925.5</td>
      <td>420</td>
      <td>3.77857</td>
      <td>1587</td>
    </tr>
    <tr>
      <th>4</th>
      <td>60565</td>
      <td>2842</td>
      <td>3.94053</td>
      <td>11199</td>
      <td>2829</td>
      <td>3.92824</td>
      <td>11113</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Determine Total Review Count and Rating "Wins" by City (Winner Take All)
combined_data["Rating Wins"] = np.where(combined_data["Mexican Average Rating"] > combined_data["Italian Average Rating"], "Mexican", "Italian")
combined_data["Review Count Wins"] = np.where(combined_data["Mexican Review Count"] > combined_data["Italian Review Count"], "Mexican", "Italian")
```


```python
# View Combined Data
combined_data.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Zip Code</th>
      <th>Mexican Review Count</th>
      <th>Mexican Average Rating</th>
      <th>Mexican Weighted Rating</th>
      <th>Italian Review Count</th>
      <th>Italian Average Rating</th>
      <th>Italian Weighted Rating</th>
      <th>Rating Wins</th>
      <th>Review Count Wins</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>76556</td>
      <td>97</td>
      <td>4.1134</td>
      <td>399</td>
      <td>63</td>
      <td>3.78571</td>
      <td>238.5</td>
      <td>Mexican</td>
      <td>Mexican</td>
    </tr>
    <tr>
      <th>1</th>
      <td>72039</td>
      <td>256</td>
      <td>4.11133</td>
      <td>1052.5</td>
      <td>266</td>
      <td>3.81955</td>
      <td>1016</td>
      <td>Mexican</td>
      <td>Italian</td>
    </tr>
    <tr>
      <th>2</th>
      <td>61606</td>
      <td>378</td>
      <td>3.64286</td>
      <td>1377</td>
      <td>66</td>
      <td>3.2197</td>
      <td>212.5</td>
      <td>Mexican</td>
      <td>Mexican</td>
    </tr>
    <tr>
      <th>3</th>
      <td>47232</td>
      <td>222</td>
      <td>4.16892</td>
      <td>925.5</td>
      <td>420</td>
      <td>3.77857</td>
      <td>1587</td>
      <td>Mexican</td>
      <td>Italian</td>
    </tr>
    <tr>
      <th>4</th>
      <td>60565</td>
      <td>2842</td>
      <td>3.94053</td>
      <td>11199</td>
      <td>2829</td>
      <td>3.92824</td>
      <td>11113</td>
      <td>Mexican</td>
      <td>Mexican</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Tally number of cities where one food variety wins on ratings over the other
combined_data["Rating Wins"].value_counts()
```




    Mexican    273
    Italian    245
    Name: Rating Wins, dtype: int64




```python
# Tally number of cities where one food variety wins on review counts over the other
combined_data["Review Count Wins"].value_counts()
```




    Italian    298
    Mexican    220
    Name: Review Count Wins, dtype: int64



## Display Summary of Results


```python
# Model 1: Head-to-Head Review Counts
italian_summary = pd.DataFrame({"Review Counts": italian_data["Italian Review Count"].sum(),
                                "Rating Average": italian_data["Italian Average Rating"].mean(),
                                "Review Count Wins": combined_data["Review Count Wins"].value_counts()["Italian"],
                                "Rating Wins": combined_data["Rating Wins"].value_counts()["Italian"]}, index=["Italian"])

mexican_summary = pd.DataFrame({"Review Counts": mexican_data["Mexican Review Count"].sum(),
                                "Rating Average": mexican_data["Mexican Average Rating"].mean(),
                                "Review Count Wins": combined_data["Review Count Wins"].value_counts()["Mexican"],
                                "Rating Wins": combined_data["Rating Wins"].value_counts()["Mexican"]}, index=["Mexican"])

final_summary = pd.concat([mexican_summary, italian_summary])
final_summary
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Rating Average</th>
      <th>Rating Wins</th>
      <th>Review Count Wins</th>
      <th>Review Counts</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Mexican</th>
      <td>3.826588</td>
      <td>273</td>
      <td>220</td>
      <td>476889</td>
    </tr>
    <tr>
      <th>Italian</th>
      <td>3.806869</td>
      <td>245</td>
      <td>298</td>
      <td>573733</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Plot Rating Average
plt.clf()
final_summary["Rating Average"].plot.bar()
plt.title("Yelp Ratings by Food Variety")
plt.ylabel("Average Rating")
plt.xlabel("Food Variety")
plt.xticks(rotation="horizontal")
plt.grid(True)
plt.show()
```


![png](output_25_0.png)



```python
# Plot Rating Wins
plt.clf()
final_summary["Rating Wins"].plot.bar()
plt.title("# of Zip Codes with Preference by Food Variety According to Rating")
plt.ylabel("Number of Zip Codes")
plt.xlabel("Food Variety")
plt.xticks(rotation="horizontal")
plt.grid(True)
plt.show()
```


![png](output_26_0.png)



```python
# Plot Review Count
plt.clf()
final_summary["Review Counts"].plot.bar()
plt.title("Yelp Review Counts by Food Variety")
plt.ylabel("Reviwe Counts")
plt.xlabel("Food Variety")
plt.xticks(rotation="horizontal")
plt.grid(True)
plt.show()
```


![png](output_27_0.png)



```python
# Plot Review Count
plt.clf()
final_summary["Review Count Wins"].plot.bar()
plt.title("# of Zip Codes with Preference by Food Variety According to Review Counts")
plt.ylabel("Number of Zip Codes")
plt.xlabel("Food Variety")
plt.xticks(rotation="horizontal")
plt.grid(True)
plt.show()
```


![png](output_28_0.png)



```python
# Histogram Italian Food (Ratings)
plt.figure()

# Subplot 1 (Italian)
plt.subplot(121)
combined_data["Italian Average Rating"].plot.hist(bins=[0, 0.5, 1, 1.5, 2, 2.5, 3, 3.5, 4, 4.5, 5.0], color="blue", alpha=0.6)
plt.xlabel("Italian Restaurant Ratings")
plt.xlim([1, 5.0])
plt.ylim([0, 400])

# Subplot 2 (Mexican)
plt.subplot(122)
combined_data["Mexican Average Rating"].plot.hist(bins=[0, 0.5, 1, 1.5, 2, 2.5, 3, 3.5, 4, 4.5, 5.0], color="red", alpha=0.6)
plt.xlabel("Mexican Retaurant Ratings")
plt.xlim([1, 5.0])
plt.ylim([0, 400])

# Show Plot
plt.show()
```


![png](output_29_0.png)


## Statistical Analysis


```python
# Run a t-test on average rating and number of reviewers
mexican_ratings = combined_data["Mexican Average Rating"]
italian_ratings = combined_data["Italian Average Rating"]

mexican_review_counts = combined_data["Mexican Review Count"]
italian_review_counts = combined_data["Italian Review Count"]
```


```python
# Run T-Test on Ratings
ttest_ind(mexican_ratings.values, italian_ratings.values, equal_var=False)
```




    Ttest_indResult(statistic=1.0702777109866186, pvalue=0.28474610492878616)




```python
# Run T-Test on Review Counts
ttest_ind(mexican_review_counts.values, italian_review_counts.values, equal_var=False)
```




    Ttest_indResult(statistic=-1.9031271166792225, pvalue=0.057326353522100949)



## Conclusions
---
Based on our analysis, it is clear that the American preference for Italian and Mexican food are similar in nature. As a whole, Americans rate Mexican and Italian restaurants at statistically similar scores (Avg score: 3.8, p-value: 0.285). However, there  exists statistically significant evidence that Americans write more reviews of Italian restaurants than Mexican. This may indicate that there is an increased interest in visiting Italian restaurants at an experiential level. However, it may also merely suggest that Yelp users enjoy writing reviews on Italian restaurants more than Mexican restaurants.

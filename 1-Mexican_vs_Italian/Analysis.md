
# Italian vs. Mexican Food
---

The below script provides an analytic approach for assessing the American preference of Italian vs. Mexican food. Using data from the US Census and the Yelp API, the script randomly selects over 500 zip codes and aggregates the reviews of the 20 most popular Italian and Mexican restaurants in each area. Summary data is then reported using Python Pandas. 


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

# Yelp API Key
ykey_id = "1GwZyE0zIjSujpHtlMnodQ"
ykey_secret = "mcTmghB48JIH0xoNWLldvsX9uIiOLQfdi0gR8LWdFt02lboCAF9vxSSd1MI0KtZ0"
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
# Sell all zip codes with a population over 1000 from a set of randomly selected list of 700 zip code locations 
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
      <th>621</th>
      <td>29468</td>
      <td>Pineville, SC 29468, USA</td>
      <td>1768</td>
      <td>43.8</td>
      <td>19663.0</td>
      <td>28526</td>
    </tr>
    <tr>
      <th>226</th>
      <td>25414</td>
      <td>Charles Town, WV 25414, USA</td>
      <td>17147</td>
      <td>40.1</td>
      <td>72833.0</td>
      <td>32308</td>
    </tr>
    <tr>
      <th>233</th>
      <td>93545</td>
      <td>Lone Pine, CA 93545, USA</td>
      <td>2214</td>
      <td>40.6</td>
      <td>32473.0</td>
      <td>18444</td>
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
    <tr>
      <th>235</th>
      <td>3064</td>
      <td>Nashua, NH 03064, USA</td>
      <td>14533</td>
      <td>40.6</td>
      <td>64026.0</td>
      <td>34045</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Show the total number of zip codes that met our population cut-off
selected_zips.count()
```




    Zipcode              521
    Address              521
    Population           521
    Median Age           521
    Household Income     520
    Per Capita Income    521
    dtype: int64




```python
# Show the average population of our representive sample set
selected_zips["Population"].mean()
```




    13860.940499040307




```python
# Show the average population of our representive sample set
selected_zips["Household Income"].mean()
```




    56293.278846153844




```python
# Show the average population of our representive sample set
selected_zips["Median Age"].mean()
```




    40.02053742802301



## Yelp Data Retrieval


```python
# Create Two DataFrames to store the Italian and the Mexican Data 
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
    https://api.yelp.com/v3/businesses/search?term=Italian&location=29468
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=29468
    2
    https://api.yelp.com/v3/businesses/search?term=Italian&location=25414
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25414
    3
    https://api.yelp.com/v3/businesses/search?term=Italian&location=93545
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=93545
    4
    https://api.yelp.com/v3/businesses/search?term=Italian&location=60565
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60565
    5
    https://api.yelp.com/v3/businesses/search?term=Italian&location=3064
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=3064
    6
    https://api.yelp.com/v3/businesses/search?term=Italian&location=38049
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=38049
    7
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28529
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28529
    8
    https://api.yelp.com/v3/businesses/search?term=Italian&location=93428
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=93428
    9
    https://api.yelp.com/v3/businesses/search?term=Italian&location=41564
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=41564
    10
    https://api.yelp.com/v3/businesses/search?term=Italian&location=98106
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98106
    11
    https://api.yelp.com/v3/businesses/search?term=Italian&location=73014
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=73014
    12
    https://api.yelp.com/v3/businesses/search?term=Italian&location=72039
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72039
    13
    https://api.yelp.com/v3/businesses/search?term=Italian&location=38224
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=38224
    14
    https://api.yelp.com/v3/businesses/search?term=Italian&location=92392
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92392
    15
    https://api.yelp.com/v3/businesses/search?term=Italian&location=11220
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=11220
    16
    https://api.yelp.com/v3/businesses/search?term=Italian&location=57369
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57369
    17
    https://api.yelp.com/v3/businesses/search?term=Italian&location=31771
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=31771
    18
    https://api.yelp.com/v3/businesses/search?term=Italian&location=47436
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47436
    19
    https://api.yelp.com/v3/businesses/search?term=Italian&location=40769
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=40769
    20
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49048
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49048
    21
    https://api.yelp.com/v3/businesses/search?term=Italian&location=30666
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30666
    22
    https://api.yelp.com/v3/businesses/search?term=Italian&location=71046
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71046
    23
    https://api.yelp.com/v3/businesses/search?term=Italian&location=84116
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=84116
    24
    https://api.yelp.com/v3/businesses/search?term=Italian&location=46312
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=46312
    25
    https://api.yelp.com/v3/businesses/search?term=Italian&location=30177
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30177
    26
    https://api.yelp.com/v3/businesses/search?term=Italian&location=5454
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=5454
    27
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28618
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28618
    28
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27358
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27358
    29
    https://api.yelp.com/v3/businesses/search?term=Italian&location=65017
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=65017
    30
    https://api.yelp.com/v3/businesses/search?term=Italian&location=69130
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=69130
    31
    https://api.yelp.com/v3/businesses/search?term=Italian&location=29834
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=29834
    32
    https://api.yelp.com/v3/businesses/search?term=Italian&location=45005
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=45005
    33
    https://api.yelp.com/v3/businesses/search?term=Italian&location=97229
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97229
    34
    https://api.yelp.com/v3/businesses/search?term=Italian&location=6422
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6422
    35
    https://api.yelp.com/v3/businesses/search?term=Italian&location=58503
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=58503
    36
    https://api.yelp.com/v3/businesses/search?term=Italian&location=36701
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=36701
    37
    https://api.yelp.com/v3/businesses/search?term=Italian&location=13637
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13637
    38
    https://api.yelp.com/v3/businesses/search?term=Italian&location=76706
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76706
    39
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49112
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49112
    40
    https://api.yelp.com/v3/businesses/search?term=Italian&location=18914
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=18914
    41
    https://api.yelp.com/v3/businesses/search?term=Italian&location=3104
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=3104
    42
    https://api.yelp.com/v3/businesses/search?term=Italian&location=42031
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=42031
    43
    https://api.yelp.com/v3/businesses/search?term=Italian&location=85606
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85606
    44
    https://api.yelp.com/v3/businesses/search?term=Italian&location=10803
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=10803
    45
    https://api.yelp.com/v3/businesses/search?term=Italian&location=7603
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=7603
    46
    https://api.yelp.com/v3/businesses/search?term=Italian&location=78941
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=78941
    47
    https://api.yelp.com/v3/businesses/search?term=Italian&location=25571
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25571
    48
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28208
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28208
    49
    https://api.yelp.com/v3/businesses/search?term=Italian&location=78004
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=78004
    50
    https://api.yelp.com/v3/businesses/search?term=Italian&location=85233
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85233
    51
    https://api.yelp.com/v3/businesses/search?term=Italian&location=76556
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76556
    52
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28436
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28436
    53
    https://api.yelp.com/v3/businesses/search?term=Italian&location=13827
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13827
    54
    https://api.yelp.com/v3/businesses/search?term=Italian&location=48084
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48084
    55
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44629
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44629
    56
    https://api.yelp.com/v3/businesses/search?term=Italian&location=69334
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=69334
    57
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49098
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49098
    58
    https://api.yelp.com/v3/businesses/search?term=Italian&location=36558
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=36558
    59
    https://api.yelp.com/v3/businesses/search?term=Italian&location=92843
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92843
    60
    https://api.yelp.com/v3/businesses/search?term=Italian&location=23146
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=23146
    61
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49781
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49781
    62
    https://api.yelp.com/v3/businesses/search?term=Italian&location=17737
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17737
    63
    https://api.yelp.com/v3/businesses/search?term=Italian&location=97146
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97146
    64
    https://api.yelp.com/v3/businesses/search?term=Italian&location=62330
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=62330
    65
    https://api.yelp.com/v3/businesses/search?term=Italian&location=10007
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=10007
    66
    https://api.yelp.com/v3/businesses/search?term=Italian&location=22191
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22191
    67
    https://api.yelp.com/v3/businesses/search?term=Italian&location=1527
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=1527
    68
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44305
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44305
    69
    https://api.yelp.com/v3/businesses/search?term=Italian&location=624
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=624
    70
    https://api.yelp.com/v3/businesses/search?term=Italian&location=58369
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=58369
    71
    https://api.yelp.com/v3/businesses/search?term=Italian&location=17980
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17980
    72
    https://api.yelp.com/v3/businesses/search?term=Italian&location=43050
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=43050
    73
    https://api.yelp.com/v3/businesses/search?term=Italian&location=13068
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13068
    74
    https://api.yelp.com/v3/businesses/search?term=Italian&location=11105
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=11105
    75
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28697
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28697
    76
    https://api.yelp.com/v3/businesses/search?term=Italian&location=47232
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47232
    77
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27822
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27822
    78
    https://api.yelp.com/v3/businesses/search?term=Italian&location=20886
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=20886
    79
    https://api.yelp.com/v3/businesses/search?term=Italian&location=60433
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60433
    80
    https://api.yelp.com/v3/businesses/search?term=Italian&location=17774
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17774
    81
    https://api.yelp.com/v3/businesses/search?term=Italian&location=60622
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60622
    82
    https://api.yelp.com/v3/businesses/search?term=Italian&location=83638
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=83638
    83
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55396
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55396
    84
    https://api.yelp.com/v3/businesses/search?term=Italian&location=99737
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=99737
    Uh oh
    85
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55735
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55735
    86
    https://api.yelp.com/v3/businesses/search?term=Italian&location=80831
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=80831
    87
    https://api.yelp.com/v3/businesses/search?term=Italian&location=66614
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=66614
    88
    https://api.yelp.com/v3/businesses/search?term=Italian&location=1585
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=1585
    89
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55342
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55342
    90
    https://api.yelp.com/v3/businesses/search?term=Italian&location=65243
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=65243
    91
    https://api.yelp.com/v3/businesses/search?term=Italian&location=41046
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=41046
    92
    https://api.yelp.com/v3/businesses/search?term=Italian&location=30564
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30564
    93
    https://api.yelp.com/v3/businesses/search?term=Italian&location=90003
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=90003
    94
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55060
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55060
    95
    https://api.yelp.com/v3/businesses/search?term=Italian&location=54151
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54151
    96
    https://api.yelp.com/v3/businesses/search?term=Italian&location=30309
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30309
    97
    https://api.yelp.com/v3/businesses/search?term=Italian&location=35005
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35005
    98
    https://api.yelp.com/v3/businesses/search?term=Italian&location=67661
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=67661
    99
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44093
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44093
    100
    https://api.yelp.com/v3/businesses/search?term=Italian&location=45780
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=45780
    101
    https://api.yelp.com/v3/businesses/search?term=Italian&location=69101
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=69101
    102
    https://api.yelp.com/v3/businesses/search?term=Italian&location=78414
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=78414
    103
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55435
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55435
    104
    https://api.yelp.com/v3/businesses/search?term=Italian&location=85553
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85553
    105
    https://api.yelp.com/v3/businesses/search?term=Italian&location=36619
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=36619
    106
    https://api.yelp.com/v3/businesses/search?term=Italian&location=77031
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77031
    107
    https://api.yelp.com/v3/businesses/search?term=Italian&location=60305
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60305
    108
    https://api.yelp.com/v3/businesses/search?term=Italian&location=62449
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=62449
    109
    https://api.yelp.com/v3/businesses/search?term=Italian&location=18237
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=18237
    110
    https://api.yelp.com/v3/businesses/search?term=Italian&location=48851
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48851
    111
    https://api.yelp.com/v3/businesses/search?term=Italian&location=48469
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48469
    112
    https://api.yelp.com/v3/businesses/search?term=Italian&location=77373
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77373
    113
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49616
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49616
    114
    https://api.yelp.com/v3/businesses/search?term=Italian&location=16153
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=16153
    115
    https://api.yelp.com/v3/businesses/search?term=Italian&location=34224
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=34224
    116
    https://api.yelp.com/v3/businesses/search?term=Italian&location=42025
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=42025
    117
    https://api.yelp.com/v3/businesses/search?term=Italian&location=57105
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57105
    118
    https://api.yelp.com/v3/businesses/search?term=Italian&location=22642
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22642
    119
    https://api.yelp.com/v3/businesses/search?term=Italian&location=71405
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71405
    120
    https://api.yelp.com/v3/businesses/search?term=Italian&location=50201
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=50201
    121
    https://api.yelp.com/v3/businesses/search?term=Italian&location=12010
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12010
    122
    https://api.yelp.com/v3/businesses/search?term=Italian&location=85353
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85353
    123
    https://api.yelp.com/v3/businesses/search?term=Italian&location=25442
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25442
    124
    https://api.yelp.com/v3/businesses/search?term=Italian&location=17018
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17018
    125
    https://api.yelp.com/v3/businesses/search?term=Italian&location=25401
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25401
    126
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27298
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27298
    127
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37345
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37345
    128
    https://api.yelp.com/v3/businesses/search?term=Italian&location=22102
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22102
    129
    https://api.yelp.com/v3/businesses/search?term=Italian&location=48386
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48386
    130
    https://api.yelp.com/v3/businesses/search?term=Italian&location=35801
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35801
    131
    https://api.yelp.com/v3/businesses/search?term=Italian&location=75783
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=75783
    132
    https://api.yelp.com/v3/businesses/search?term=Italian&location=72722
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72722
    133
    https://api.yelp.com/v3/businesses/search?term=Italian&location=76950
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76950
    134
    https://api.yelp.com/v3/businesses/search?term=Italian&location=61062
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=61062
    135
    https://api.yelp.com/v3/businesses/search?term=Italian&location=22066
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22066
    136
    https://api.yelp.com/v3/businesses/search?term=Italian&location=52141
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=52141
    137
    https://api.yelp.com/v3/businesses/search?term=Italian&location=12941
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12941
    138
    https://api.yelp.com/v3/businesses/search?term=Italian&location=77807
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77807
    139
    https://api.yelp.com/v3/businesses/search?term=Italian&location=72007
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72007
    140
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28635
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28635
    141
    https://api.yelp.com/v3/businesses/search?term=Italian&location=64081
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=64081
    142
    https://api.yelp.com/v3/businesses/search?term=Italian&location=11385
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=11385
    143
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27283
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27283
    144
    https://api.yelp.com/v3/businesses/search?term=Italian&location=66507
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=66507
    145
    https://api.yelp.com/v3/businesses/search?term=Italian&location=21901
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=21901
    146
    https://api.yelp.com/v3/businesses/search?term=Italian&location=48468
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48468
    147
    https://api.yelp.com/v3/businesses/search?term=Italian&location=97016
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97016
    148
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55063
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55063
    149
    https://api.yelp.com/v3/businesses/search?term=Italian&location=68317
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=68317
    150
    https://api.yelp.com/v3/businesses/search?term=Italian&location=71353
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71353
    151
    https://api.yelp.com/v3/businesses/search?term=Italian&location=57078
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57078
    152
    https://api.yelp.com/v3/businesses/search?term=Italian&location=21053
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=21053
    153
    https://api.yelp.com/v3/businesses/search?term=Italian&location=730
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=730
    154
    https://api.yelp.com/v3/businesses/search?term=Italian&location=76525
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76525
    155
    https://api.yelp.com/v3/businesses/search?term=Italian&location=10553
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=10553
    156
    https://api.yelp.com/v3/businesses/search?term=Italian&location=35956
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35956
    157
    https://api.yelp.com/v3/businesses/search?term=Italian&location=62025
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=62025
    158
    https://api.yelp.com/v3/businesses/search?term=Italian&location=85250
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85250
    159
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55346
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55346
    160
    https://api.yelp.com/v3/businesses/search?term=Italian&location=23832
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=23832
    161
    https://api.yelp.com/v3/businesses/search?term=Italian&location=72434
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72434
    162
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44273
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44273
    163
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44313
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44313
    164
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37101
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37101
    165
    https://api.yelp.com/v3/businesses/search?term=Italian&location=63138
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=63138
    166
    https://api.yelp.com/v3/businesses/search?term=Italian&location=757
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=757
    167
    https://api.yelp.com/v3/businesses/search?term=Italian&location=54022
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54022
    168
    https://api.yelp.com/v3/businesses/search?term=Italian&location=29001
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=29001
    169
    https://api.yelp.com/v3/businesses/search?term=Italian&location=75640
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=75640
    170
    https://api.yelp.com/v3/businesses/search?term=Italian&location=54229
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54229
    171
    https://api.yelp.com/v3/businesses/search?term=Italian&location=7724
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=7724
    172
    https://api.yelp.com/v3/businesses/search?term=Italian&location=48416
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48416
    173
    https://api.yelp.com/v3/businesses/search?term=Italian&location=76031
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76031
    174
    https://api.yelp.com/v3/businesses/search?term=Italian&location=6019
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6019
    175
    https://api.yelp.com/v3/businesses/search?term=Italian&location=4622
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=4622
    176
    https://api.yelp.com/v3/businesses/search?term=Italian&location=30120
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30120
    177
    https://api.yelp.com/v3/businesses/search?term=Italian&location=22027
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22027
    178
    https://api.yelp.com/v3/businesses/search?term=Italian&location=23228
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=23228
    179
    https://api.yelp.com/v3/businesses/search?term=Italian&location=85383
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85383
    180
    https://api.yelp.com/v3/businesses/search?term=Italian&location=92115
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92115
    181
    https://api.yelp.com/v3/businesses/search?term=Italian&location=98270
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98270
    182
    https://api.yelp.com/v3/businesses/search?term=Italian&location=63128
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=63128
    183
    https://api.yelp.com/v3/businesses/search?term=Italian&location=68005
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=68005
    184
    https://api.yelp.com/v3/businesses/search?term=Italian&location=6062
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6062
    185
    https://api.yelp.com/v3/businesses/search?term=Italian&location=33542
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33542
    186
    https://api.yelp.com/v3/businesses/search?term=Italian&location=17938
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17938
    187
    https://api.yelp.com/v3/businesses/search?term=Italian&location=21061
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=21061
    188
    https://api.yelp.com/v3/businesses/search?term=Italian&location=38229
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=38229
    189
    https://api.yelp.com/v3/businesses/search?term=Italian&location=95237
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=95237
    190
    https://api.yelp.com/v3/businesses/search?term=Italian&location=92252
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92252
    191
    https://api.yelp.com/v3/businesses/search?term=Italian&location=61270
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=61270
    192
    https://api.yelp.com/v3/businesses/search?term=Italian&location=76599
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76599
    193
    https://api.yelp.com/v3/businesses/search?term=Italian&location=7606
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=7606
    194
    https://api.yelp.com/v3/businesses/search?term=Italian&location=85262
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85262
    195
    https://api.yelp.com/v3/businesses/search?term=Italian&location=24137
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=24137
    196
    https://api.yelp.com/v3/businesses/search?term=Italian&location=22834
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22834
    197
    https://api.yelp.com/v3/businesses/search?term=Italian&location=8872
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=8872
    198
    https://api.yelp.com/v3/businesses/search?term=Italian&location=1430
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=1430
    199
    https://api.yelp.com/v3/businesses/search?term=Italian&location=52533
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=52533
    200
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49080
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49080
    201
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37336
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37336
    202
    https://api.yelp.com/v3/businesses/search?term=Italian&location=48659
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48659
    203
    https://api.yelp.com/v3/businesses/search?term=Italian&location=54822
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54822
    204
    https://api.yelp.com/v3/businesses/search?term=Italian&location=21750
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=21750
    205
    https://api.yelp.com/v3/businesses/search?term=Italian&location=84020
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=84020
    206
    https://api.yelp.com/v3/businesses/search?term=Italian&location=97875
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97875
    207
    https://api.yelp.com/v3/businesses/search?term=Italian&location=34209
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=34209
    208
    https://api.yelp.com/v3/businesses/search?term=Italian&location=77880
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77880
    209
    https://api.yelp.com/v3/businesses/search?term=Italian&location=71340
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71340
    210
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49234
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49234
    211
    https://api.yelp.com/v3/businesses/search?term=Italian&location=71834
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71834
    212
    https://api.yelp.com/v3/businesses/search?term=Italian&location=78611
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=78611
    213
    https://api.yelp.com/v3/businesses/search?term=Italian&location=83634
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=83634
    214
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49058
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49058
    215
    https://api.yelp.com/v3/businesses/search?term=Italian&location=5261
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=5261
    216
    https://api.yelp.com/v3/businesses/search?term=Italian&location=4107
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=4107
    217
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27317
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27317
    218
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44147
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44147
    219
    https://api.yelp.com/v3/businesses/search?term=Italian&location=23602
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=23602
    220
    https://api.yelp.com/v3/businesses/search?term=Italian&location=96783
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=96783
    221
    https://api.yelp.com/v3/businesses/search?term=Italian&location=20634
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=20634
    222
    https://api.yelp.com/v3/businesses/search?term=Italian&location=692
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=692
    223
    https://api.yelp.com/v3/businesses/search?term=Italian&location=31007
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=31007
    224
    https://api.yelp.com/v3/businesses/search?term=Italian&location=35907
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35907
    225
    https://api.yelp.com/v3/businesses/search?term=Italian&location=97633
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97633
    226
    https://api.yelp.com/v3/businesses/search?term=Italian&location=47393
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47393
    227
    https://api.yelp.com/v3/businesses/search?term=Italian&location=94556
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=94556
    228
    https://api.yelp.com/v3/businesses/search?term=Italian&location=98362
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98362
    229
    https://api.yelp.com/v3/businesses/search?term=Italian&location=63039
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=63039
    230
    https://api.yelp.com/v3/businesses/search?term=Italian&location=77336
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77336
    231
    https://api.yelp.com/v3/businesses/search?term=Italian&location=53118
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=53118
    232
    https://api.yelp.com/v3/businesses/search?term=Italian&location=57315
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57315
    233
    https://api.yelp.com/v3/businesses/search?term=Italian&location=32535
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=32535
    234
    https://api.yelp.com/v3/businesses/search?term=Italian&location=5156
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=5156
    235
    https://api.yelp.com/v3/businesses/search?term=Italian&location=92111
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92111
    236
    https://api.yelp.com/v3/businesses/search?term=Italian&location=34744
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=34744
    237
    https://api.yelp.com/v3/businesses/search?term=Italian&location=53156
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=53156
    238
    https://api.yelp.com/v3/businesses/search?term=Italian&location=62613
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=62613
    239
    https://api.yelp.com/v3/businesses/search?term=Italian&location=84118
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=84118
    240
    https://api.yelp.com/v3/businesses/search?term=Italian&location=76638
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76638
    241
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44510
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44510
    242
    https://api.yelp.com/v3/businesses/search?term=Italian&location=91207
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=91207
    243
    https://api.yelp.com/v3/businesses/search?term=Italian&location=48221
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48221
    244
    https://api.yelp.com/v3/businesses/search?term=Italian&location=29113
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=29113
    245
    https://api.yelp.com/v3/businesses/search?term=Italian&location=17051
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17051
    246
    https://api.yelp.com/v3/businesses/search?term=Italian&location=23665
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=23665
    247
    https://api.yelp.com/v3/businesses/search?term=Italian&location=53576
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=53576
    248
    https://api.yelp.com/v3/businesses/search?term=Italian&location=99661
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=99661
    Uh oh
    249
    https://api.yelp.com/v3/businesses/search?term=Italian&location=25559
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25559
    250
    https://api.yelp.com/v3/businesses/search?term=Italian&location=50525
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=50525
    251
    https://api.yelp.com/v3/businesses/search?term=Italian&location=17327
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17327
    252
    https://api.yelp.com/v3/businesses/search?term=Italian&location=43804
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=43804
    253
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27549
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27549
    254
    https://api.yelp.com/v3/businesses/search?term=Italian&location=79764
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=79764
    255
    https://api.yelp.com/v3/businesses/search?term=Italian&location=12723
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12723
    256
    https://api.yelp.com/v3/businesses/search?term=Italian&location=67839
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=67839
    257
    https://api.yelp.com/v3/businesses/search?term=Italian&location=12804
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12804
    258
    https://api.yelp.com/v3/businesses/search?term=Italian&location=2740
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=2740
    259
    https://api.yelp.com/v3/businesses/search?term=Italian&location=24083
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=24083
    260
    https://api.yelp.com/v3/businesses/search?term=Italian&location=50543
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=50543
    261
    https://api.yelp.com/v3/businesses/search?term=Italian&location=87528
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=87528
    262
    https://api.yelp.com/v3/businesses/search?term=Italian&location=19076
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=19076
    263
    https://api.yelp.com/v3/businesses/search?term=Italian&location=10303
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=10303
    264
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28076
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28076
    265
    https://api.yelp.com/v3/businesses/search?term=Italian&location=86507
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=86507
    266
    https://api.yelp.com/v3/businesses/search?term=Italian&location=98409
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98409
    267
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55433
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55433
    268
    https://api.yelp.com/v3/businesses/search?term=Italian&location=95490
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=95490
    269
    https://api.yelp.com/v3/businesses/search?term=Italian&location=42717
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=42717
    270
    https://api.yelp.com/v3/businesses/search?term=Italian&location=82932
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=82932
    271
    https://api.yelp.com/v3/businesses/search?term=Italian&location=96818
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=96818
    272
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28395
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28395
    273
    https://api.yelp.com/v3/businesses/search?term=Italian&location=33614
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33614
    274
    https://api.yelp.com/v3/businesses/search?term=Italian&location=29170
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=29170
    275
    https://api.yelp.com/v3/businesses/search?term=Italian&location=6069
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6069
    276
    https://api.yelp.com/v3/businesses/search?term=Italian&location=90302
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=90302
    277
    https://api.yelp.com/v3/businesses/search?term=Italian&location=14715
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14715
    278
    https://api.yelp.com/v3/businesses/search?term=Italian&location=47003
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47003
    279
    https://api.yelp.com/v3/businesses/search?term=Italian&location=95240
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=95240
    280
    https://api.yelp.com/v3/businesses/search?term=Italian&location=50315
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=50315
    281
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27541
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27541
    282
    https://api.yelp.com/v3/businesses/search?term=Italian&location=42456
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=42456
    283
    https://api.yelp.com/v3/businesses/search?term=Italian&location=38870
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=38870
    284
    https://api.yelp.com/v3/businesses/search?term=Italian&location=75454
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=75454
    285
    https://api.yelp.com/v3/businesses/search?term=Italian&location=98012
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98012
    286
    https://api.yelp.com/v3/businesses/search?term=Italian&location=89316
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=89316
    Uh oh
    287
    https://api.yelp.com/v3/businesses/search?term=Italian&location=46236
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=46236
    288
    https://api.yelp.com/v3/businesses/search?term=Italian&location=23219
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=23219
    289
    https://api.yelp.com/v3/businesses/search?term=Italian&location=59752
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=59752
    290
    https://api.yelp.com/v3/businesses/search?term=Italian&location=71031
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71031
    291
    https://api.yelp.com/v3/businesses/search?term=Italian&location=46996
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=46996
    292
    https://api.yelp.com/v3/businesses/search?term=Italian&location=16930
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=16930
    293
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49874
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49874
    294
    https://api.yelp.com/v3/businesses/search?term=Italian&location=60645
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60645
    295
    https://api.yelp.com/v3/businesses/search?term=Italian&location=6883
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6883
    296
    https://api.yelp.com/v3/businesses/search?term=Italian&location=13326
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13326
    297
    https://api.yelp.com/v3/businesses/search?term=Italian&location=30601
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30601
    298
    https://api.yelp.com/v3/businesses/search?term=Italian&location=33129
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33129
    299
    https://api.yelp.com/v3/businesses/search?term=Italian&location=33559
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33559
    300
    https://api.yelp.com/v3/businesses/search?term=Italian&location=48191
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48191
    301
    https://api.yelp.com/v3/businesses/search?term=Italian&location=47591
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47591
    302
    https://api.yelp.com/v3/businesses/search?term=Italian&location=6516
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6516
    303
    https://api.yelp.com/v3/businesses/search?term=Italian&location=77378
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77378
    304
    https://api.yelp.com/v3/businesses/search?term=Italian&location=75457
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=75457
    305
    https://api.yelp.com/v3/businesses/search?term=Italian&location=42366
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=42366
    306
    https://api.yelp.com/v3/businesses/search?term=Italian&location=97396
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97396
    307
    https://api.yelp.com/v3/businesses/search?term=Italian&location=98051
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98051
    308
    https://api.yelp.com/v3/businesses/search?term=Italian&location=69347
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=69347
    309
    https://api.yelp.com/v3/businesses/search?term=Italian&location=4010
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=4010
    310
    https://api.yelp.com/v3/businesses/search?term=Italian&location=75149
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=75149
    311
    https://api.yelp.com/v3/businesses/search?term=Italian&location=36736
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=36736
    312
    https://api.yelp.com/v3/businesses/search?term=Italian&location=94040
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=94040
    313
    https://api.yelp.com/v3/businesses/search?term=Italian&location=98358
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98358
    314
    https://api.yelp.com/v3/businesses/search?term=Italian&location=14807
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14807
    315
    https://api.yelp.com/v3/businesses/search?term=Italian&location=26807
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=26807
    316
    https://api.yelp.com/v3/businesses/search?term=Italian&location=33496
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33496
    317
    https://api.yelp.com/v3/businesses/search?term=Italian&location=1469
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=1469
    318
    https://api.yelp.com/v3/businesses/search?term=Italian&location=2717
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=2717
    319
    https://api.yelp.com/v3/businesses/search?term=Italian&location=7718
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=7718
    320
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55802
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55802
    321
    https://api.yelp.com/v3/businesses/search?term=Italian&location=4268
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=4268
    322
    https://api.yelp.com/v3/businesses/search?term=Italian&location=77328
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77328
    323
    https://api.yelp.com/v3/businesses/search?term=Italian&location=79118
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=79118
    324
    https://api.yelp.com/v3/businesses/search?term=Italian&location=8540
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=8540
    325
    https://api.yelp.com/v3/businesses/search?term=Italian&location=10913
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=10913
    326
    https://api.yelp.com/v3/businesses/search?term=Italian&location=60018
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60018
    327
    https://api.yelp.com/v3/businesses/search?term=Italian&location=10309
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=10309
    328
    https://api.yelp.com/v3/businesses/search?term=Italian&location=11722
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=11722
    329
    https://api.yelp.com/v3/businesses/search?term=Italian&location=59522
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=59522
    Uh oh
    330
    https://api.yelp.com/v3/businesses/search?term=Italian&location=83849
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=83849
    331
    https://api.yelp.com/v3/businesses/search?term=Italian&location=32064
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=32064
    332
    https://api.yelp.com/v3/businesses/search?term=Italian&location=58647
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=58647
    333
    https://api.yelp.com/v3/businesses/search?term=Italian&location=84093
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=84093
    334
    https://api.yelp.com/v3/businesses/search?term=Italian&location=78214
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=78214
    335
    https://api.yelp.com/v3/businesses/search?term=Italian&location=30152
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30152
    336
    https://api.yelp.com/v3/businesses/search?term=Italian&location=95008
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=95008
    337
    https://api.yelp.com/v3/businesses/search?term=Italian&location=63935
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=63935
    338
    https://api.yelp.com/v3/businesses/search?term=Italian&location=2056
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=2056
    339
    https://api.yelp.com/v3/businesses/search?term=Italian&location=59201
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=59201
    340
    https://api.yelp.com/v3/businesses/search?term=Italian&location=57004
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57004
    341
    https://api.yelp.com/v3/businesses/search?term=Italian&location=70748
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=70748
    342
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37743
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37743
    343
    https://api.yelp.com/v3/businesses/search?term=Italian&location=5658
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=5658
    344
    https://api.yelp.com/v3/businesses/search?term=Italian&location=957
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=957
    345
    https://api.yelp.com/v3/businesses/search?term=Italian&location=3077
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=3077
    346
    https://api.yelp.com/v3/businesses/search?term=Italian&location=38858
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=38858
    347
    https://api.yelp.com/v3/businesses/search?term=Italian&location=32187
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=32187
    348
    https://api.yelp.com/v3/businesses/search?term=Italian&location=59846
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=59846
    349
    https://api.yelp.com/v3/businesses/search?term=Italian&location=97389
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97389
    350
    https://api.yelp.com/v3/businesses/search?term=Italian&location=46524
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=46524
    351
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28782
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28782
    352
    https://api.yelp.com/v3/businesses/search?term=Italian&location=53588
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=53588
    353
    https://api.yelp.com/v3/businesses/search?term=Italian&location=11693
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=11693
    354
    https://api.yelp.com/v3/businesses/search?term=Italian&location=38504
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=38504
    355
    https://api.yelp.com/v3/businesses/search?term=Italian&location=13335
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13335
    356
    https://api.yelp.com/v3/businesses/search?term=Italian&location=47932
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47932
    357
    https://api.yelp.com/v3/businesses/search?term=Italian&location=98394
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98394
    358
    https://api.yelp.com/v3/businesses/search?term=Italian&location=67835
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=67835
    359
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44708
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44708
    360
    https://api.yelp.com/v3/businesses/search?term=Italian&location=7082
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=7082
    361
    https://api.yelp.com/v3/businesses/search?term=Italian&location=10984
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=10984
    362
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44276
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44276
    363
    https://api.yelp.com/v3/businesses/search?term=Italian&location=72833
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72833
    364
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28226
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28226
    365
    https://api.yelp.com/v3/businesses/search?term=Italian&location=25865
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25865
    366
    https://api.yelp.com/v3/businesses/search?term=Italian&location=98591
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98591
    367
    https://api.yelp.com/v3/businesses/search?term=Italian&location=3048
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=3048
    368
    https://api.yelp.com/v3/businesses/search?term=Italian&location=79703
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=79703
    369
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37191
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37191
    370
    https://api.yelp.com/v3/businesses/search?term=Italian&location=90210
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=90210
    371
    https://api.yelp.com/v3/businesses/search?term=Italian&location=35951
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35951
    372
    https://api.yelp.com/v3/businesses/search?term=Italian&location=46617
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=46617
    373
    https://api.yelp.com/v3/businesses/search?term=Italian&location=95246
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=95246
    374
    https://api.yelp.com/v3/businesses/search?term=Italian&location=39180
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=39180
    375
    https://api.yelp.com/v3/businesses/search?term=Italian&location=97321
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97321
    376
    https://api.yelp.com/v3/businesses/search?term=Italian&location=53936
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=53936
    377
    https://api.yelp.com/v3/businesses/search?term=Italian&location=71957
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71957
    378
    https://api.yelp.com/v3/businesses/search?term=Italian&location=67530
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=67530
    379
    https://api.yelp.com/v3/businesses/search?term=Italian&location=54979
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54979
    380
    https://api.yelp.com/v3/businesses/search?term=Italian&location=17214
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17214
    381
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49639
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49639
    382
    https://api.yelp.com/v3/businesses/search?term=Italian&location=95051
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=95051
    383
    https://api.yelp.com/v3/businesses/search?term=Italian&location=61604
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=61604
    384
    https://api.yelp.com/v3/businesses/search?term=Italian&location=14131
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14131
    385
    https://api.yelp.com/v3/businesses/search?term=Italian&location=12831
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12831
    386
    https://api.yelp.com/v3/businesses/search?term=Italian&location=60468
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60468
    387
    https://api.yelp.com/v3/businesses/search?term=Italian&location=46323
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=46323
    388
    https://api.yelp.com/v3/businesses/search?term=Italian&location=57005
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57005
    389
    https://api.yelp.com/v3/businesses/search?term=Italian&location=45036
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=45036
    390
    https://api.yelp.com/v3/businesses/search?term=Italian&location=73951
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=73951
    391
    https://api.yelp.com/v3/businesses/search?term=Italian&location=5483
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=5483
    392
    https://api.yelp.com/v3/businesses/search?term=Italian&location=50002
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=50002
    393
    https://api.yelp.com/v3/businesses/search?term=Italian&location=14806
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14806
    394
    https://api.yelp.com/v3/businesses/search?term=Italian&location=47431
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47431
    395
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27262
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27262
    396
    https://api.yelp.com/v3/businesses/search?term=Italian&location=45684
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=45684
    397
    https://api.yelp.com/v3/businesses/search?term=Italian&location=15656
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=15656
    398
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44867
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44867
    399
    https://api.yelp.com/v3/businesses/search?term=Italian&location=30268
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30268
    400
    https://api.yelp.com/v3/businesses/search?term=Italian&location=35117
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35117
    401
    https://api.yelp.com/v3/businesses/search?term=Italian&location=26205
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=26205
    402
    https://api.yelp.com/v3/businesses/search?term=Italian&location=54733
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54733
    403
    https://api.yelp.com/v3/businesses/search?term=Italian&location=2382
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=2382
    404
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37764
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37764
    405
    https://api.yelp.com/v3/businesses/search?term=Italian&location=57785
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57785
    406
    https://api.yelp.com/v3/businesses/search?term=Italian&location=24281
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=24281
    407
    https://api.yelp.com/v3/businesses/search?term=Italian&location=16870
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=16870
    408
    https://api.yelp.com/v3/businesses/search?term=Italian&location=6801
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6801
    409
    https://api.yelp.com/v3/businesses/search?term=Italian&location=13402
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13402
    410
    https://api.yelp.com/v3/businesses/search?term=Italian&location=32409
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=32409
    411
    https://api.yelp.com/v3/businesses/search?term=Italian&location=32025
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=32025
    412
    https://api.yelp.com/v3/businesses/search?term=Italian&location=12148
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12148
    413
    https://api.yelp.com/v3/businesses/search?term=Italian&location=14033
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14033
    414
    https://api.yelp.com/v3/businesses/search?term=Italian&location=77520
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77520
    415
    https://api.yelp.com/v3/businesses/search?term=Italian&location=32444
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=32444
    416
    https://api.yelp.com/v3/businesses/search?term=Italian&location=22025
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22025
    417
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37080
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37080
    418
    https://api.yelp.com/v3/businesses/search?term=Italian&location=12789
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12789
    419
    https://api.yelp.com/v3/businesses/search?term=Italian&location=33029
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33029
    420
    https://api.yelp.com/v3/businesses/search?term=Italian&location=31405
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=31405
    421
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49269
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49269
    422
    https://api.yelp.com/v3/businesses/search?term=Italian&location=33025
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33025
    423
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37737
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37737
    424
    https://api.yelp.com/v3/businesses/search?term=Italian&location=76135
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76135
    425
    https://api.yelp.com/v3/businesses/search?term=Italian&location=15206
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=15206
    426
    https://api.yelp.com/v3/businesses/search?term=Italian&location=79416
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=79416
    427
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49415
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49415
    428
    https://api.yelp.com/v3/businesses/search?term=Italian&location=8234
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=8234
    429
    https://api.yelp.com/v3/businesses/search?term=Italian&location=56358
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=56358
    430
    https://api.yelp.com/v3/businesses/search?term=Italian&location=30236
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30236
    431
    https://api.yelp.com/v3/businesses/search?term=Italian&location=5301
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=5301
    432
    https://api.yelp.com/v3/businesses/search?term=Italian&location=3276
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=3276
    433
    https://api.yelp.com/v3/businesses/search?term=Italian&location=8002
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=8002
    434
    https://api.yelp.com/v3/businesses/search?term=Italian&location=4357
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=4357
    435
    https://api.yelp.com/v3/businesses/search?term=Italian&location=66088
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=66088
    436
    https://api.yelp.com/v3/businesses/search?term=Italian&location=47670
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47670
    437
    https://api.yelp.com/v3/businesses/search?term=Italian&location=43758
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=43758
    438
    https://api.yelp.com/v3/businesses/search?term=Italian&location=8854
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=8854
    439
    https://api.yelp.com/v3/businesses/search?term=Italian&location=72315
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72315
    440
    https://api.yelp.com/v3/businesses/search?term=Italian&location=79705
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=79705
    441
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28327
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28327
    442
    https://api.yelp.com/v3/businesses/search?term=Italian&location=19057
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=19057
    443
    https://api.yelp.com/v3/businesses/search?term=Italian&location=14555
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14555
    444
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27288
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27288
    445
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37067
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37067
    446
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37398
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37398
    447
    https://api.yelp.com/v3/businesses/search?term=Italian&location=35475
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35475
    448
    https://api.yelp.com/v3/businesses/search?term=Italian&location=83313
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=83313
    449
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28732
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28732
    450
    https://api.yelp.com/v3/businesses/search?term=Italian&location=77039
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77039
    451
    https://api.yelp.com/v3/businesses/search?term=Italian&location=98101
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98101
    452
    https://api.yelp.com/v3/businesses/search?term=Italian&location=40060
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=40060
    453
    https://api.yelp.com/v3/businesses/search?term=Italian&location=74956
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=74956
    454
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27249
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27249
    455
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37727
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37727
    456
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49036
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49036
    457
    https://api.yelp.com/v3/businesses/search?term=Italian&location=68930
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=68930
    458
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55976
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55976
    459
    https://api.yelp.com/v3/businesses/search?term=Italian&location=6824
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6824
    460
    https://api.yelp.com/v3/businesses/search?term=Italian&location=58075
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=58075
    461
    https://api.yelp.com/v3/businesses/search?term=Italian&location=25253
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25253
    462
    https://api.yelp.com/v3/businesses/search?term=Italian&location=12759
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12759
    463
    https://api.yelp.com/v3/businesses/search?term=Italian&location=63038
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=63038
    464
    https://api.yelp.com/v3/businesses/search?term=Italian&location=68147
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=68147
    465
    https://api.yelp.com/v3/businesses/search?term=Italian&location=41649
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=41649
    466
    https://api.yelp.com/v3/businesses/search?term=Italian&location=99506
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=99506
    467
    https://api.yelp.com/v3/businesses/search?term=Italian&location=22952
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22952
    468
    https://api.yelp.com/v3/businesses/search?term=Italian&location=62340
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=62340
    469
    https://api.yelp.com/v3/businesses/search?term=Italian&location=93637
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=93637
    470
    https://api.yelp.com/v3/businesses/search?term=Italian&location=4418
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=4418
    471
    https://api.yelp.com/v3/businesses/search?term=Italian&location=14411
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14411
    472
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49893
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49893
    473
    https://api.yelp.com/v3/businesses/search?term=Italian&location=85743
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85743
    474
    https://api.yelp.com/v3/businesses/search?term=Italian&location=63304
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=63304
    475
    https://api.yelp.com/v3/businesses/search?term=Italian&location=88337
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=88337
    476
    https://api.yelp.com/v3/businesses/search?term=Italian&location=44143
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44143
    477
    https://api.yelp.com/v3/businesses/search?term=Italian&location=66052
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=66052
    478
    https://api.yelp.com/v3/businesses/search?term=Italian&location=8859
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=8859
    Uh oh
    479
    https://api.yelp.com/v3/businesses/search?term=Italian&location=60091
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60091
    480
    https://api.yelp.com/v3/businesses/search?term=Italian&location=25427
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25427
    481
    https://api.yelp.com/v3/businesses/search?term=Italian&location=60137
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60137
    482
    https://api.yelp.com/v3/businesses/search?term=Italian&location=18042
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=18042
    483
    https://api.yelp.com/v3/businesses/search?term=Italian&location=2145
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=2145
    484
    https://api.yelp.com/v3/businesses/search?term=Italian&location=92543
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92543
    485
    https://api.yelp.com/v3/businesses/search?term=Italian&location=27043
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27043
    486
    https://api.yelp.com/v3/businesses/search?term=Italian&location=1922
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=1922
    487
    https://api.yelp.com/v3/businesses/search?term=Italian&location=49411
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49411
    488
    https://api.yelp.com/v3/businesses/search?term=Italian&location=72401
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72401
    489
    https://api.yelp.com/v3/businesses/search?term=Italian&location=80482
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=80482
    490
    https://api.yelp.com/v3/businesses/search?term=Italian&location=47952
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47952
    491
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28371
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28371
    492
    https://api.yelp.com/v3/businesses/search?term=Italian&location=56726
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=56726
    Uh oh
    493
    https://api.yelp.com/v3/businesses/search?term=Italian&location=20143
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=20143
    494
    https://api.yelp.com/v3/businesses/search?term=Italian&location=97366
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97366
    495
    https://api.yelp.com/v3/businesses/search?term=Italian&location=65332
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=65332
    496
    https://api.yelp.com/v3/businesses/search?term=Italian&location=14304
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14304
    497
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55069
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55069
    498
    https://api.yelp.com/v3/businesses/search?term=Italian&location=76943
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76943
    499
    https://api.yelp.com/v3/businesses/search?term=Italian&location=52320
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=52320
    500
    https://api.yelp.com/v3/businesses/search?term=Italian&location=53018
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=53018
    501
    https://api.yelp.com/v3/businesses/search?term=Italian&location=56529
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=56529
    502
    https://api.yelp.com/v3/businesses/search?term=Italian&location=59802
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=59802
    503
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28454
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28454
    504
    https://api.yelp.com/v3/businesses/search?term=Italian&location=55419
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55419
    505
    https://api.yelp.com/v3/businesses/search?term=Italian&location=77434
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77434
    506
    https://api.yelp.com/v3/businesses/search?term=Italian&location=13495
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13495
    507
    https://api.yelp.com/v3/businesses/search?term=Italian&location=28349
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28349
    508
    https://api.yelp.com/v3/businesses/search?term=Italian&location=37923
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37923
    509
    https://api.yelp.com/v3/businesses/search?term=Italian&location=19372
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=19372
    510
    https://api.yelp.com/v3/businesses/search?term=Italian&location=59820
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=59820
    511
    https://api.yelp.com/v3/businesses/search?term=Italian&location=97022
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97022
    512
    https://api.yelp.com/v3/businesses/search?term=Italian&location=60421
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60421
    513
    https://api.yelp.com/v3/businesses/search?term=Italian&location=54747
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54747
    514
    https://api.yelp.com/v3/businesses/search?term=Italian&location=93925
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=93925
    515
    https://api.yelp.com/v3/businesses/search?term=Italian&location=62063
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=62063
    516
    https://api.yelp.com/v3/businesses/search?term=Italian&location=6377
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6377
    517
    https://api.yelp.com/v3/businesses/search?term=Italian&location=18519
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=18519
    518
    https://api.yelp.com/v3/businesses/search?term=Italian&location=2169
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=2169
    519
    https://api.yelp.com/v3/businesses/search?term=Italian&location=19023
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=19023
    520
    https://api.yelp.com/v3/businesses/search?term=Italian&location=92285
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92285
    521
    https://api.yelp.com/v3/businesses/search?term=Italian&location=76310
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76310
    


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
      <th>621</th>
      <td>29468</td>
      <td>77</td>
      <td>3.35065</td>
      <td>258</td>
    </tr>
    <tr>
      <th>226</th>
      <td>25414</td>
      <td>135</td>
      <td>3.41111</td>
      <td>460.5</td>
    </tr>
    <tr>
      <th>233</th>
      <td>93545</td>
      <td>702</td>
      <td>4.10684</td>
      <td>2883</td>
    </tr>
    <tr>
      <th>67</th>
      <td>60565</td>
      <td>2829</td>
      <td>3.92807</td>
      <td>11112.5</td>
    </tr>
    <tr>
      <th>235</th>
      <td>3064</td>
      <td>1253</td>
      <td>3.77813</td>
      <td>4734</td>
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
      <th>621</th>
      <td>29468</td>
      <td>199</td>
      <td>3.8593</td>
      <td>768</td>
    </tr>
    <tr>
      <th>226</th>
      <td>25414</td>
      <td>206</td>
      <td>4.15534</td>
      <td>856</td>
    </tr>
    <tr>
      <th>233</th>
      <td>93545</td>
      <td>213</td>
      <td>3.64554</td>
      <td>776.5</td>
    </tr>
    <tr>
      <th>67</th>
      <td>60565</td>
      <td>2836</td>
      <td>3.94059</td>
      <td>11175.5</td>
    </tr>
    <tr>
      <th>235</th>
      <td>3064</td>
      <td>544</td>
      <td>3.72518</td>
      <td>2026.5</td>
    </tr>
  </tbody>
</table>
</div>



## Calculate Summaries


```python
mexican_data["Mexican Review Count"].sum()
```




    469100




```python
italian_data["Italian Review Count"].sum()
```




    561872




```python
mexican_data["Mexican Weighted Rating"].sum() / mexican_data["Mexican Review Count"].sum()
```




    3.905826049882754




```python
italian_data["Italian Weighted Rating"].sum() / italian_data["Italian Review Count"].sum()
```




    3.9436802332203778




```python
# Combine Data Frames into a single Data Frame
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
      <td>29468</td>
      <td>199</td>
      <td>3.8593</td>
      <td>768</td>
      <td>77</td>
      <td>3.35065</td>
      <td>258</td>
    </tr>
    <tr>
      <th>1</th>
      <td>25414</td>
      <td>206</td>
      <td>4.15534</td>
      <td>856</td>
      <td>135</td>
      <td>3.41111</td>
      <td>460.5</td>
    </tr>
    <tr>
      <th>2</th>
      <td>93545</td>
      <td>213</td>
      <td>3.64554</td>
      <td>776.5</td>
      <td>702</td>
      <td>4.10684</td>
      <td>2883</td>
    </tr>
    <tr>
      <th>3</th>
      <td>60565</td>
      <td>2836</td>
      <td>3.94059</td>
      <td>11175.5</td>
      <td>2829</td>
      <td>3.92807</td>
      <td>11112.5</td>
    </tr>
    <tr>
      <th>4</th>
      <td>3064</td>
      <td>544</td>
      <td>3.72518</td>
      <td>2026.5</td>
      <td>1253</td>
      <td>3.77813</td>
      <td>4734</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Total Rating and Popularity "Wins"
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
      <td>29468</td>
      <td>199</td>
      <td>3.8593</td>
      <td>768</td>
      <td>77</td>
      <td>3.35065</td>
      <td>258</td>
      <td>Mexican</td>
      <td>Mexican</td>
    </tr>
    <tr>
      <th>1</th>
      <td>25414</td>
      <td>206</td>
      <td>4.15534</td>
      <td>856</td>
      <td>135</td>
      <td>3.41111</td>
      <td>460.5</td>
      <td>Mexican</td>
      <td>Mexican</td>
    </tr>
    <tr>
      <th>2</th>
      <td>93545</td>
      <td>213</td>
      <td>3.64554</td>
      <td>776.5</td>
      <td>702</td>
      <td>4.10684</td>
      <td>2883</td>
      <td>Italian</td>
      <td>Italian</td>
    </tr>
    <tr>
      <th>3</th>
      <td>60565</td>
      <td>2836</td>
      <td>3.94059</td>
      <td>11175.5</td>
      <td>2829</td>
      <td>3.92807</td>
      <td>11112.5</td>
      <td>Mexican</td>
      <td>Mexican</td>
    </tr>
    <tr>
      <th>4</th>
      <td>3064</td>
      <td>544</td>
      <td>3.72518</td>
      <td>2026.5</td>
      <td>1253</td>
      <td>3.77813</td>
      <td>4734</td>
      <td>Italian</td>
      <td>Italian</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Tally number of cities where one type wins on ratings over the other
combined_data["Rating Wins"].value_counts()
```




    Mexican    267
    Italian    248
    Name: Rating Wins, dtype: int64




```python
# Tally number of cities where one type wins on review counts over the other
combined_data["Review Count Wins"].value_counts()
```




    Italian    298
    Mexican    217
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
      <td>3.825374</td>
      <td>267</td>
      <td>217</td>
      <td>469100</td>
    </tr>
    <tr>
      <th>Italian</th>
      <td>3.807411</td>
      <td>248</td>
      <td>298</td>
      <td>561872</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Plot Rating Average
plt.clf()
final_summary["Rating Average"].plot.bar()
plt.show()
```


![png](output_25_0.png)



```python
# Plot Rating Wins
plt.clf()
final_summary["Rating Wins"].plot.bar()
plt.show()
```


![png](output_26_0.png)



```python
# Plot Review Count
plt.clf()
final_summary["Review Counts"].plot.bar()
plt.show()
```


![png](output_27_0.png)



```python
# Plot Review Count
plt.clf()
final_summary["Review Count Wins"].plot.bar()
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
ttest_ind(mexican_ratings.values, italian_ratings.values)
```




    Ttest_indResult(statistic=0.97343025910497216, pvalue=0.33056849735046245)




```python
# Run T-Test on Review Counts
ttest_ind(mexican_review_counts.values, italian_review_counts.values)
```




    Ttest_indResult(statistic=-1.8606196373601354, pvalue=0.063083275022233751)



## Conclusions
---
Based on our analysis, it is clear that American preference for Italian and Mexican food are very similar in nature. As a whole, Americans rate Mexican and Italian restaurants at statistically similar scores. However, there does exist evidence that Americans do write more reviews on Italian restaurants. This may indicate that there is an increased interest in visiting Italian restaurants at an experiential level. (However, this data may also merely suggest that Yelp users happen to enjoy writing reviews on Italian restaurants more).


```python

```

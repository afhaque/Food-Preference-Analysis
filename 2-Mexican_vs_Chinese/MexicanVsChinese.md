
# Chinese vs. Mexican Food
---

The below script provides an analytic approach for assessing the American preference of Chinese vs. Mexican food. Using data from the US Census and the Yelp API, the script randomly selects over 500 zip codes and aggregates the reviews of the 20 most popular chinese and Mexican restaurants in each area. Summary data is then reported using Python Pandas. 


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
      <th>74</th>
      <td>31771</td>
      <td>Norman Park, GA 31771, USA</td>
      <td>6207</td>
      <td>32.9</td>
      <td>33524.0</td>
      <td>17690</td>
    </tr>
    <tr>
      <th>398</th>
      <td>47952</td>
      <td>Kingman, IN 47952, USA</td>
      <td>2685</td>
      <td>45.5</td>
      <td>37232.0</td>
      <td>20271</td>
    </tr>
    <tr>
      <th>579</th>
      <td>74956</td>
      <td>Shady Point, OK 74956, USA</td>
      <td>2269</td>
      <td>36.4</td>
      <td>40652.0</td>
      <td>18796</td>
    </tr>
    <tr>
      <th>590</th>
      <td>33129</td>
      <td>Miami, FL 33129, USA</td>
      <td>14198</td>
      <td>38.9</td>
      <td>54127.0</td>
      <td>41352</td>
    </tr>
    <tr>
      <th>562</th>
      <td>16930</td>
      <td>Liberty, PA 16930, USA</td>
      <td>1348</td>
      <td>41.0</td>
      <td>44934.0</td>
      <td>20611</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Show the total number of zip codes that met our population cut-off
selected_zips.count()
```




    Zipcode              520
    Address              520
    Population           520
    Median Age           520
    Household Income     519
    Per Capita Income    520
    dtype: int64




```python
# Show the average population of our representive sample set
selected_zips["Population"].mean()
```




    13966.148076923077




```python
# Show the average population of our representive sample set
selected_zips["Household Income"].mean()
```




    55928.131021194604




```python
# Show the average population of our representive sample set
selected_zips["Median Age"].mean()
```




    39.943846153846195



## Yelp Data Retrieval


```python
# Create Two DataFrames to store the chinese andMexican Data 
chinese_data = pd.DataFrame();
mexican_data = pd.DataFrame();

# Setup the DataFrames to have appropriate columns
chinese_data["Zip Code"] = ""
chinese_data["chinese Review Count"] = ""
chinese_data["chinese Average Rating"] = ""
chinese_data["chinese Weighted Rating"] = ""

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
    target_url_chinese = "https://api.yelp.com/v3/businesses/search?term=chinese&location=%s" % (row["Zipcode"])
    target_url_mexican = "https://api.yelp.com/v3/businesses/search?term=Mexican&location=%s" % (row["Zipcode"])
    
    # Print the URLs to ensure logging
    print(counter)
    print(target_url_chinese)
    print(target_url_mexican)
    
    # Get the Yelp Reviews
    yelp_reviews_chinese = requests.get(target_url_chinese, headers=headers).json()
    yelp_reviews_mexican = requests.get(target_url_mexican, headers=headers).json()
    
    # Calculate the total reviews and weighted rankings
    chinese_review_count = 0
    chinese_weighted_review = 0
    mexican_review_count = 0
    mexican_weighted_review = 0
    
    # Use Try-Except to handle errors
    try:
        
        # Loop through all records to calculate the review count and weighted review value
        for business in yelp_reviews_chinese["businesses"]:

            chinese_review_count = chinese_review_count + business["review_count"]
            chinese_weighted_review = chinese_weighted_review + business["review_count"] * business["rating"]

        for business in yelp_reviews_mexican["businesses"]:
            mexican_review_count = mexican_review_count + business["review_count"]
            mexican_weighted_review = mexican_weighted_review + business["review_count"] * business["rating"] 
        
        # Append the data to the appropriate column of the data frames
        chinese_data.set_value(index, "Zip Code", row["Zipcode"])
        chinese_data.set_value(index, "chinese Review Count", chinese_review_count)
        chinese_data.set_value(index, "chinese Average Rating", chinese_weighted_review / chinese_review_count)
        chinese_data.set_value(index, "chinese Weighted Rating", chinese_weighted_review)

        mexican_data.set_value(index, "Zip Code", row["Zipcode"])
        mexican_data.set_value(index, "Mexican Review Count", mexican_review_count)
        mexican_data.set_value(index, "Mexican Average Rating", mexican_weighted_review / mexican_review_count)
        mexican_data.set_value(index, "Mexican Weighted Rating", mexican_weighted_review)

    except:
        print("Uh oh")
        
```

    1
    https://api.yelp.com/v3/businesses/search?term=chinese&location=31771
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=31771
    2
    https://api.yelp.com/v3/businesses/search?term=chinese&location=47952
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47952
    3
    https://api.yelp.com/v3/businesses/search?term=chinese&location=74956
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=74956
    4
    https://api.yelp.com/v3/businesses/search?term=chinese&location=33129
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33129
    5
    https://api.yelp.com/v3/businesses/search?term=chinese&location=16930
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=16930
    6
    https://api.yelp.com/v3/businesses/search?term=chinese&location=44093
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44093
    7
    https://api.yelp.com/v3/businesses/search?term=chinese&location=76031
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76031
    8
    https://api.yelp.com/v3/businesses/search?term=chinese&location=60421
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60421
    9
    https://api.yelp.com/v3/businesses/search?term=chinese&location=98362
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98362
    10
    https://api.yelp.com/v3/businesses/search?term=chinese&location=49415
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49415
    11
    https://api.yelp.com/v3/businesses/search?term=chinese&location=85743
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85743
    12
    https://api.yelp.com/v3/businesses/search?term=chinese&location=28732
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28732
    13
    https://api.yelp.com/v3/businesses/search?term=chinese&location=27298
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27298
    14
    https://api.yelp.com/v3/businesses/search?term=chinese&location=72039
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72039
    15
    https://api.yelp.com/v3/businesses/search?term=chinese&location=8872
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=8872
    16
    https://api.yelp.com/v3/businesses/search?term=chinese&location=53118
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=53118
    17
    https://api.yelp.com/v3/businesses/search?term=chinese&location=46996
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=46996
    18
    https://api.yelp.com/v3/businesses/search?term=chinese&location=27249
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27249
    19
    https://api.yelp.com/v3/businesses/search?term=chinese&location=62330
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=62330
    20
    https://api.yelp.com/v3/businesses/search?term=chinese&location=77434
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77434
    21
    https://api.yelp.com/v3/businesses/search?term=chinese&location=90210
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=90210
    22
    https://api.yelp.com/v3/businesses/search?term=chinese&location=77880
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77880
    23
    https://api.yelp.com/v3/businesses/search?term=chinese&location=11105
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=11105
    24
    https://api.yelp.com/v3/businesses/search?term=chinese&location=4010
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=4010
    25
    https://api.yelp.com/v3/businesses/search?term=chinese&location=50201
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=50201
    26
    https://api.yelp.com/v3/businesses/search?term=chinese&location=2740
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=2740
    27
    https://api.yelp.com/v3/businesses/search?term=chinese&location=730
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=730
    28
    https://api.yelp.com/v3/businesses/search?term=chinese&location=26807
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=26807
    29
    https://api.yelp.com/v3/businesses/search?term=chinese&location=42456
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=42456
    30
    https://api.yelp.com/v3/businesses/search?term=chinese&location=30120
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30120
    31
    https://api.yelp.com/v3/businesses/search?term=chinese&location=53588
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=53588
    32
    https://api.yelp.com/v3/businesses/search?term=chinese&location=76135
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76135
    33
    https://api.yelp.com/v3/businesses/search?term=chinese&location=28697
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28697
    34
    https://api.yelp.com/v3/businesses/search?term=chinese&location=10913
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=10913
    35
    https://api.yelp.com/v3/businesses/search?term=chinese&location=28454
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28454
    36
    https://api.yelp.com/v3/businesses/search?term=chinese&location=60565
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60565
    37
    https://api.yelp.com/v3/businesses/search?term=chinese&location=23146
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=23146
    38
    https://api.yelp.com/v3/businesses/search?term=chinese&location=28436
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28436
    39
    https://api.yelp.com/v3/businesses/search?term=chinese&location=98106
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98106
    40
    https://api.yelp.com/v3/businesses/search?term=chinese&location=38858
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=38858
    41
    https://api.yelp.com/v3/businesses/search?term=chinese&location=49874
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49874
    42
    https://api.yelp.com/v3/businesses/search?term=chinese&location=35956
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35956
    43
    https://api.yelp.com/v3/businesses/search?term=chinese&location=30309
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30309
    44
    https://api.yelp.com/v3/businesses/search?term=chinese&location=76638
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76638
    45
    https://api.yelp.com/v3/businesses/search?term=chinese&location=8002
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=8002
    46
    https://api.yelp.com/v3/businesses/search?term=chinese&location=19023
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=19023
    47
    https://api.yelp.com/v3/businesses/search?term=chinese&location=37080
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37080
    48
    https://api.yelp.com/v3/businesses/search?term=chinese&location=54822
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54822
    49
    https://api.yelp.com/v3/businesses/search?term=chinese&location=62449
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=62449
    50
    https://api.yelp.com/v3/businesses/search?term=chinese&location=10007
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=10007
    51
    https://api.yelp.com/v3/businesses/search?term=chinese&location=24281
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=24281
    52
    https://api.yelp.com/v3/businesses/search?term=chinese&location=37737
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37737
    53
    https://api.yelp.com/v3/businesses/search?term=chinese&location=38870
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=38870
    54
    https://api.yelp.com/v3/businesses/search?term=chinese&location=72315
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72315
    55
    https://api.yelp.com/v3/businesses/search?term=chinese&location=73014
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=73014
    56
    https://api.yelp.com/v3/businesses/search?term=chinese&location=20886
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=20886
    57
    https://api.yelp.com/v3/businesses/search?term=chinese&location=56529
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=56529
    58
    https://api.yelp.com/v3/businesses/search?term=chinese&location=84118
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=84118
    59
    https://api.yelp.com/v3/businesses/search?term=chinese&location=5301
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=5301
    60
    https://api.yelp.com/v3/businesses/search?term=chinese&location=12789
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12789
    61
    https://api.yelp.com/v3/businesses/search?term=chinese&location=98591
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98591
    62
    https://api.yelp.com/v3/businesses/search?term=chinese&location=13827
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13827
    63
    https://api.yelp.com/v3/businesses/search?term=chinese&location=2145
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=2145
    64
    https://api.yelp.com/v3/businesses/search?term=chinese&location=2056
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=2056
    65
    https://api.yelp.com/v3/businesses/search?term=chinese&location=98051
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98051
    66
    https://api.yelp.com/v3/businesses/search?term=chinese&location=46236
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=46236
    67
    https://api.yelp.com/v3/businesses/search?term=chinese&location=92115
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92115
    68
    https://api.yelp.com/v3/businesses/search?term=chinese&location=6062
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6062
    69
    https://api.yelp.com/v3/businesses/search?term=chinese&location=17938
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17938
    70
    https://api.yelp.com/v3/businesses/search?term=chinese&location=57785
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57785
    71
    https://api.yelp.com/v3/businesses/search?term=chinese&location=47431
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47431
    72
    https://api.yelp.com/v3/businesses/search?term=chinese&location=10309
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=10309
    73
    https://api.yelp.com/v3/businesses/search?term=chinese&location=2382
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=2382
    74
    https://api.yelp.com/v3/businesses/search?term=chinese&location=72434
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72434
    75
    https://api.yelp.com/v3/businesses/search?term=chinese&location=66052
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=66052
    76
    https://api.yelp.com/v3/businesses/search?term=chinese&location=49781
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49781
    77
    https://api.yelp.com/v3/businesses/search?term=chinese&location=11220
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=11220
    78
    https://api.yelp.com/v3/businesses/search?term=chinese&location=44629
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44629
    79
    https://api.yelp.com/v3/businesses/search?term=chinese&location=77039
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77039
    80
    https://api.yelp.com/v3/businesses/search?term=chinese&location=98012
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98012
    81
    https://api.yelp.com/v3/businesses/search?term=chinese&location=86507
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=86507
    82
    https://api.yelp.com/v3/businesses/search?term=chinese&location=55396
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55396
    83
    https://api.yelp.com/v3/businesses/search?term=chinese&location=30268
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30268
    84
    https://api.yelp.com/v3/businesses/search?term=chinese&location=25414
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25414
    85
    https://api.yelp.com/v3/businesses/search?term=chinese&location=63039
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=63039
    86
    https://api.yelp.com/v3/businesses/search?term=chinese&location=85383
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85383
    87
    https://api.yelp.com/v3/businesses/search?term=chinese&location=20143
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=20143
    88
    https://api.yelp.com/v3/businesses/search?term=chinese&location=55342
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55342
    89
    https://api.yelp.com/v3/businesses/search?term=chinese&location=98358
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98358
    90
    https://api.yelp.com/v3/businesses/search?term=chinese&location=3104
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=3104
    91
    https://api.yelp.com/v3/businesses/search?term=chinese&location=71353
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71353
    92
    https://api.yelp.com/v3/businesses/search?term=chinese&location=63128
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=63128
    93
    https://api.yelp.com/v3/businesses/search?term=chinese&location=63038
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=63038
    94
    https://api.yelp.com/v3/businesses/search?term=chinese&location=13495
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13495
    95
    https://api.yelp.com/v3/businesses/search?term=chinese&location=28371
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28371
    96
    https://api.yelp.com/v3/businesses/search?term=chinese&location=32409
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=32409
    97
    https://api.yelp.com/v3/businesses/search?term=chinese&location=65243
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=65243
    98
    https://api.yelp.com/v3/businesses/search?term=chinese&location=82932
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=82932
    99
    https://api.yelp.com/v3/businesses/search?term=chinese&location=92392
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92392
    100
    https://api.yelp.com/v3/businesses/search?term=chinese&location=44708
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44708
    101
    https://api.yelp.com/v3/businesses/search?term=chinese&location=49269
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49269
    102
    https://api.yelp.com/v3/businesses/search?term=chinese&location=11722
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=11722
    103
    https://api.yelp.com/v3/businesses/search?term=chinese&location=92843
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92843
    104
    https://api.yelp.com/v3/businesses/search?term=chinese&location=13402
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13402
    105
    https://api.yelp.com/v3/businesses/search?term=chinese&location=35951
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35951
    106
    https://api.yelp.com/v3/businesses/search?term=chinese&location=14555
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14555
    107
    https://api.yelp.com/v3/businesses/search?term=chinese&location=68005
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=68005
    108
    https://api.yelp.com/v3/businesses/search?term=chinese&location=56358
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=56358
    109
    https://api.yelp.com/v3/businesses/search?term=chinese&location=75640
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=75640
    110
    https://api.yelp.com/v3/businesses/search?term=chinese&location=95051
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=95051
    111
    https://api.yelp.com/v3/businesses/search?term=chinese&location=41046
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=41046
    112
    https://api.yelp.com/v3/businesses/search?term=chinese&location=76950
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76950
    113
    https://api.yelp.com/v3/businesses/search?term=chinese&location=98101
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98101
    114
    https://api.yelp.com/v3/businesses/search?term=chinese&location=25865
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25865
    115
    https://api.yelp.com/v3/businesses/search?term=chinese&location=38224
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=38224
    116
    https://api.yelp.com/v3/businesses/search?term=chinese&location=55435
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55435
    117
    https://api.yelp.com/v3/businesses/search?term=chinese&location=83638
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=83638
    118
    https://api.yelp.com/v3/businesses/search?term=chinese&location=47393
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47393
    119
    https://api.yelp.com/v3/businesses/search?term=chinese&location=38049
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=38049
    120
    https://api.yelp.com/v3/businesses/search?term=chinese&location=28208
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28208
    121
    https://api.yelp.com/v3/businesses/search?term=chinese&location=48191
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48191
    122
    https://api.yelp.com/v3/businesses/search?term=chinese&location=34744
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=34744
    123
    https://api.yelp.com/v3/businesses/search?term=chinese&location=98409
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98409
    124
    https://api.yelp.com/v3/businesses/search?term=chinese&location=63138
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=63138
    125
    https://api.yelp.com/v3/businesses/search?term=chinese&location=28076
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28076
    126
    https://api.yelp.com/v3/businesses/search?term=chinese&location=30236
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30236
    127
    https://api.yelp.com/v3/businesses/search?term=chinese&location=21053
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=21053
    128
    https://api.yelp.com/v3/businesses/search?term=chinese&location=44273
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44273
    129
    https://api.yelp.com/v3/businesses/search?term=chinese&location=75783
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=75783
    130
    https://api.yelp.com/v3/businesses/search?term=chinese&location=1469
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=1469
    131
    https://api.yelp.com/v3/businesses/search?term=chinese&location=48221
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48221
    132
    https://api.yelp.com/v3/businesses/search?term=chinese&location=13637
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13637
    133
    https://api.yelp.com/v3/businesses/search?term=chinese&location=90302
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=90302
    134
    https://api.yelp.com/v3/businesses/search?term=chinese&location=44313
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44313
    135
    https://api.yelp.com/v3/businesses/search?term=chinese&location=96783
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=96783
    136
    https://api.yelp.com/v3/businesses/search?term=chinese&location=37345
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37345
    137
    https://api.yelp.com/v3/businesses/search?term=chinese&location=29834
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=29834
    138
    https://api.yelp.com/v3/businesses/search?term=chinese&location=54022
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54022
    139
    https://api.yelp.com/v3/businesses/search?term=chinese&location=78214
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=78214
    140
    https://api.yelp.com/v3/businesses/search?term=chinese&location=94556
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=94556
    141
    https://api.yelp.com/v3/businesses/search?term=chinese&location=49080
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49080
    142
    https://api.yelp.com/v3/businesses/search?term=chinese&location=97633
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97633
    143
    https://api.yelp.com/v3/businesses/search?term=chinese&location=58075
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=58075
    144
    https://api.yelp.com/v3/businesses/search?term=chinese&location=36619
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=36619
    145
    https://api.yelp.com/v3/businesses/search?term=chinese&location=15206
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=15206
    146
    https://api.yelp.com/v3/businesses/search?term=chinese&location=74066
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=74066
    147
    https://api.yelp.com/v3/businesses/search?term=chinese&location=7724
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=7724
    148
    https://api.yelp.com/v3/businesses/search?term=chinese&location=85233
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85233
    149
    https://api.yelp.com/v3/businesses/search?term=chinese&location=28618
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28618
    150
    https://api.yelp.com/v3/businesses/search?term=chinese&location=21750
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=21750
    151
    https://api.yelp.com/v3/businesses/search?term=chinese&location=53936
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=53936
    152
    https://api.yelp.com/v3/businesses/search?term=chinese&location=38504
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=38504
    153
    https://api.yelp.com/v3/businesses/search?term=chinese&location=62340
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=62340
    154
    https://api.yelp.com/v3/businesses/search?term=chinese&location=31007
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=31007
    155
    https://api.yelp.com/v3/businesses/search?term=chinese&location=90003
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=90003
    156
    https://api.yelp.com/v3/businesses/search?term=chinese&location=48659
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48659
    157
    https://api.yelp.com/v3/businesses/search?term=chinese&location=24137
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=24137
    158
    https://api.yelp.com/v3/businesses/search?term=chinese&location=29001
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=29001
    159
    https://api.yelp.com/v3/businesses/search?term=chinese&location=4418
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=4418
    160
    https://api.yelp.com/v3/businesses/search?term=chinese&location=68317
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=68317
    161
    https://api.yelp.com/v3/businesses/search?term=chinese&location=30177
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30177
    162
    https://api.yelp.com/v3/businesses/search?term=chinese&location=6883
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6883
    163
    https://api.yelp.com/v3/businesses/search?term=chinese&location=5483
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=5483
    164
    https://api.yelp.com/v3/businesses/search?term=chinese&location=55346
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55346
    165
    https://api.yelp.com/v3/businesses/search?term=chinese&location=14304
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14304
    166
    https://api.yelp.com/v3/businesses/search?term=chinese&location=99506
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=99506
    167
    https://api.yelp.com/v3/businesses/search?term=chinese&location=54151
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54151
    168
    https://api.yelp.com/v3/businesses/search?term=chinese&location=50002
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=50002
    169
    https://api.yelp.com/v3/businesses/search?term=chinese&location=37067
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37067
    170
    https://api.yelp.com/v3/businesses/search?term=chinese&location=29170
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=29170
    171
    https://api.yelp.com/v3/businesses/search?term=chinese&location=44867
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44867
    172
    https://api.yelp.com/v3/businesses/search?term=chinese&location=66614
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=66614
    173
    https://api.yelp.com/v3/businesses/search?term=chinese&location=7718
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=7718
    174
    https://api.yelp.com/v3/businesses/search?term=chinese&location=54733
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54733
    175
    https://api.yelp.com/v3/businesses/search?term=chinese&location=3064
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=3064
    176
    https://api.yelp.com/v3/businesses/search?term=chinese&location=49639
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49639
    177
    https://api.yelp.com/v3/businesses/search?term=chinese&location=89316
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=89316
    Uh oh
    178
    https://api.yelp.com/v3/businesses/search?term=chinese&location=85262
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85262
    179
    https://api.yelp.com/v3/businesses/search?term=chinese&location=44143
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44143
    180
    https://api.yelp.com/v3/businesses/search?term=chinese&location=73951
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=73951
    181
    https://api.yelp.com/v3/businesses/search?term=chinese&location=75457
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=75457
    182
    https://api.yelp.com/v3/businesses/search?term=chinese&location=32064
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=32064
    183
    https://api.yelp.com/v3/businesses/search?term=chinese&location=97875
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97875
    184
    https://api.yelp.com/v3/businesses/search?term=chinese&location=60137
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60137
    185
    https://api.yelp.com/v3/businesses/search?term=chinese&location=72722
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72722
    186
    https://api.yelp.com/v3/businesses/search?term=chinese&location=97229
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97229
    187
    https://api.yelp.com/v3/businesses/search?term=chinese&location=44147
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44147
    188
    https://api.yelp.com/v3/businesses/search?term=chinese&location=92285
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92285
    189
    https://api.yelp.com/v3/businesses/search?term=chinese&location=34224
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=34224
    190
    https://api.yelp.com/v3/businesses/search?term=chinese&location=95246
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=95246
    191
    https://api.yelp.com/v3/businesses/search?term=chinese&location=957
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=957
    192
    https://api.yelp.com/v3/businesses/search?term=chinese&location=25442
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25442
    193
    https://api.yelp.com/v3/businesses/search?term=chinese&location=50315
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=50315
    194
    https://api.yelp.com/v3/businesses/search?term=chinese&location=22066
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22066
    195
    https://api.yelp.com/v3/businesses/search?term=chinese&location=53156
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=53156
    196
    https://api.yelp.com/v3/businesses/search?term=chinese&location=1527
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=1527
    197
    https://api.yelp.com/v3/businesses/search?term=chinese&location=34209
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=34209
    198
    https://api.yelp.com/v3/businesses/search?term=chinese&location=35117
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35117
    199
    https://api.yelp.com/v3/businesses/search?term=chinese&location=76310
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76310
    200
    https://api.yelp.com/v3/businesses/search?term=chinese&location=52141
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=52141
    201
    https://api.yelp.com/v3/businesses/search?term=chinese&location=57004
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57004
    202
    https://api.yelp.com/v3/businesses/search?term=chinese&location=83313
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=83313
    203
    https://api.yelp.com/v3/businesses/search?term=chinese&location=22025
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22025
    204
    https://api.yelp.com/v3/businesses/search?term=chinese&location=84093
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=84093
    205
    https://api.yelp.com/v3/businesses/search?term=chinese&location=24083
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=24083
    206
    https://api.yelp.com/v3/businesses/search?term=chinese&location=27262
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27262
    207
    https://api.yelp.com/v3/businesses/search?term=chinese&location=41649
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=41649
    208
    https://api.yelp.com/v3/businesses/search?term=chinese&location=36701
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=36701
    209
    https://api.yelp.com/v3/businesses/search?term=chinese&location=40060
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=40060
    210
    https://api.yelp.com/v3/businesses/search?term=chinese&location=55735
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55735
    211
    https://api.yelp.com/v3/businesses/search?term=chinese&location=32535
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=32535
    212
    https://api.yelp.com/v3/businesses/search?term=chinese&location=27358
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27358
    213
    https://api.yelp.com/v3/businesses/search?term=chinese&location=76599
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76599
    214
    https://api.yelp.com/v3/businesses/search?term=chinese&location=96818
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=96818
    215
    https://api.yelp.com/v3/businesses/search?term=chinese&location=23665
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=23665
    216
    https://api.yelp.com/v3/businesses/search?term=chinese&location=14411
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14411
    217
    https://api.yelp.com/v3/businesses/search?term=chinese&location=22102
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22102
    218
    https://api.yelp.com/v3/businesses/search?term=chinese&location=58369
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=58369
    219
    https://api.yelp.com/v3/businesses/search?term=chinese&location=42031
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=42031
    220
    https://api.yelp.com/v3/businesses/search?term=chinese&location=30601
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30601
    221
    https://api.yelp.com/v3/businesses/search?term=chinese&location=59846
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=59846
    222
    https://api.yelp.com/v3/businesses/search?term=chinese&location=49058
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49058
    223
    https://api.yelp.com/v3/businesses/search?term=chinese&location=69334
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=69334
    224
    https://api.yelp.com/v3/businesses/search?term=chinese&location=84020
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=84020
    225
    https://api.yelp.com/v3/businesses/search?term=chinese&location=28327
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28327
    226
    https://api.yelp.com/v3/businesses/search?term=chinese&location=78414
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=78414
    227
    https://api.yelp.com/v3/businesses/search?term=chinese&location=71405
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71405
    228
    https://api.yelp.com/v3/businesses/search?term=chinese&location=97022
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97022
    229
    https://api.yelp.com/v3/businesses/search?term=chinese&location=45036
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=45036
    230
    https://api.yelp.com/v3/businesses/search?term=chinese&location=3077
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=3077
    231
    https://api.yelp.com/v3/businesses/search?term=chinese&location=49112
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49112
    232
    https://api.yelp.com/v3/businesses/search?term=chinese&location=27822
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27822
    233
    https://api.yelp.com/v3/businesses/search?term=chinese&location=37398
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37398
    234
    https://api.yelp.com/v3/businesses/search?term=chinese&location=58503
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=58503
    235
    https://api.yelp.com/v3/businesses/search?term=chinese&location=44510
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44510
    236
    https://api.yelp.com/v3/businesses/search?term=chinese&location=97146
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97146
    237
    https://api.yelp.com/v3/businesses/search?term=chinese&location=62063
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=62063
    238
    https://api.yelp.com/v3/businesses/search?term=chinese&location=49616
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49616
    239
    https://api.yelp.com/v3/businesses/search?term=chinese&location=83634
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=83634
    240
    https://api.yelp.com/v3/businesses/search?term=chinese&location=757
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=757
    241
    https://api.yelp.com/v3/businesses/search?term=chinese&location=49098
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49098
    242
    https://api.yelp.com/v3/businesses/search?term=chinese&location=35005
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35005
    243
    https://api.yelp.com/v3/businesses/search?term=chinese&location=69347
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=69347
    244
    https://api.yelp.com/v3/businesses/search?term=chinese&location=50525
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=50525
    245
    https://api.yelp.com/v3/businesses/search?term=chinese&location=29468
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=29468
    246
    https://api.yelp.com/v3/businesses/search?term=chinese&location=67661
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=67661
    247
    https://api.yelp.com/v3/businesses/search?term=chinese&location=47932
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47932
    248
    https://api.yelp.com/v3/businesses/search?term=chinese&location=97016
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97016
    249
    https://api.yelp.com/v3/businesses/search?term=chinese&location=19372
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=19372
    250
    https://api.yelp.com/v3/businesses/search?term=chinese&location=4357
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=4357
    251
    https://api.yelp.com/v3/businesses/search?term=chinese&location=60091
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60091
    252
    https://api.yelp.com/v3/businesses/search?term=chinese&location=37923
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37923
    253
    https://api.yelp.com/v3/businesses/search?term=chinese&location=77378
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77378
    254
    https://api.yelp.com/v3/businesses/search?term=chinese&location=59201
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=59201
    255
    https://api.yelp.com/v3/businesses/search?term=chinese&location=3276
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=3276
    256
    https://api.yelp.com/v3/businesses/search?term=chinese&location=2169
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=2169
    257
    https://api.yelp.com/v3/businesses/search?term=chinese&location=28529
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28529
    258
    https://api.yelp.com/v3/businesses/search?term=chinese&location=99737
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=99737
    Uh oh
    259
    https://api.yelp.com/v3/businesses/search?term=chinese&location=6019
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6019
    260
    https://api.yelp.com/v3/businesses/search?term=chinese&location=8234
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=8234
    261
    https://api.yelp.com/v3/businesses/search?term=chinese&location=85250
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85250
    262
    https://api.yelp.com/v3/businesses/search?term=chinese&location=78611
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=78611
    263
    https://api.yelp.com/v3/businesses/search?term=chinese&location=37743
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37743
    264
    https://api.yelp.com/v3/businesses/search?term=chinese&location=60622
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60622
    265
    https://api.yelp.com/v3/businesses/search?term=chinese&location=37191
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37191
    266
    https://api.yelp.com/v3/businesses/search?term=chinese&location=21901
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=21901
    267
    https://api.yelp.com/v3/businesses/search?term=chinese&location=95008
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=95008
    268
    https://api.yelp.com/v3/businesses/search?term=chinese&location=69130
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=69130
    269
    https://api.yelp.com/v3/businesses/search?term=chinese&location=48416
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48416
    270
    https://api.yelp.com/v3/businesses/search?term=chinese&location=57005
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57005
    271
    https://api.yelp.com/v3/businesses/search?term=chinese&location=8859
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=8859
    Uh oh
    272
    https://api.yelp.com/v3/businesses/search?term=chinese&location=55063
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55063
    273
    https://api.yelp.com/v3/businesses/search?term=chinese&location=44276
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44276
    274
    https://api.yelp.com/v3/businesses/search?term=chinese&location=28395
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28395
    275
    https://api.yelp.com/v3/businesses/search?term=chinese&location=23832
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=23832
    276
    https://api.yelp.com/v3/businesses/search?term=chinese&location=48084
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48084
    277
    https://api.yelp.com/v3/businesses/search?term=chinese&location=38229
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=38229
    278
    https://api.yelp.com/v3/businesses/search?term=chinese&location=48468
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48468
    279
    https://api.yelp.com/v3/businesses/search?term=chinese&location=22834
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22834
    280
    https://api.yelp.com/v3/businesses/search?term=chinese&location=77031
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77031
    281
    https://api.yelp.com/v3/businesses/search?term=chinese&location=5156
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=5156
    282
    https://api.yelp.com/v3/businesses/search?term=chinese&location=97396
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97396
    283
    https://api.yelp.com/v3/businesses/search?term=chinese&location=37336
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37336
    284
    https://api.yelp.com/v3/businesses/search?term=chinese&location=87528
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=87528
    285
    https://api.yelp.com/v3/businesses/search?term=chinese&location=36736
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=36736
    286
    https://api.yelp.com/v3/businesses/search?term=chinese&location=47232
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47232
    287
    https://api.yelp.com/v3/businesses/search?term=chinese&location=18519
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=18519
    288
    https://api.yelp.com/v3/businesses/search?term=chinese&location=95490
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=95490
    289
    https://api.yelp.com/v3/businesses/search?term=chinese&location=33614
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33614
    290
    https://api.yelp.com/v3/businesses/search?term=chinese&location=33029
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33029
    291
    https://api.yelp.com/v3/businesses/search?term=chinese&location=62025
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=62025
    292
    https://api.yelp.com/v3/businesses/search?term=chinese&location=1585
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=1585
    293
    https://api.yelp.com/v3/businesses/search?term=chinese&location=17737
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17737
    294
    https://api.yelp.com/v3/businesses/search?term=chinese&location=60645
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60645
    295
    https://api.yelp.com/v3/businesses/search?term=chinese&location=94539
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=94539
    296
    https://api.yelp.com/v3/businesses/search?term=chinese&location=37727
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37727
    297
    https://api.yelp.com/v3/businesses/search?term=chinese&location=92252
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92252
    298
    https://api.yelp.com/v3/businesses/search?term=chinese&location=30564
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30564
    299
    https://api.yelp.com/v3/businesses/search?term=chinese&location=5658
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=5658
    300
    https://api.yelp.com/v3/businesses/search?term=chinese&location=14806
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14806
    301
    https://api.yelp.com/v3/businesses/search?term=chinese&location=72401
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72401
    302
    https://api.yelp.com/v3/businesses/search?term=chinese&location=25559
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25559
    303
    https://api.yelp.com/v3/businesses/search?term=chinese&location=84116
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=84116
    304
    https://api.yelp.com/v3/businesses/search?term=chinese&location=99661
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=99661
    Uh oh
    305
    https://api.yelp.com/v3/businesses/search?term=chinese&location=44305
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=44305
    306
    https://api.yelp.com/v3/businesses/search?term=chinese&location=69101
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=69101
    307
    https://api.yelp.com/v3/businesses/search?term=chinese&location=28349
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28349
    308
    https://api.yelp.com/v3/businesses/search?term=chinese&location=48469
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48469
    309
    https://api.yelp.com/v3/businesses/search?term=chinese&location=97389
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97389
    310
    https://api.yelp.com/v3/businesses/search?term=chinese&location=692
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=692
    311
    https://api.yelp.com/v3/businesses/search?term=chinese&location=79416
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=79416
    312
    https://api.yelp.com/v3/businesses/search?term=chinese&location=52533
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=52533
    313
    https://api.yelp.com/v3/businesses/search?term=chinese&location=67530
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=67530
    314
    https://api.yelp.com/v3/businesses/search?term=chinese&location=93428
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=93428
    315
    https://api.yelp.com/v3/businesses/search?term=chinese&location=77328
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77328
    316
    https://api.yelp.com/v3/businesses/search?term=chinese&location=35907
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35907
    317
    https://api.yelp.com/v3/businesses/search?term=chinese&location=57078
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57078
    318
    https://api.yelp.com/v3/businesses/search?term=chinese&location=78004
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=78004
    319
    https://api.yelp.com/v3/businesses/search?term=chinese&location=59752
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=59752
    320
    https://api.yelp.com/v3/businesses/search?term=chinese&location=14807
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14807
    321
    https://api.yelp.com/v3/businesses/search?term=chinese&location=25571
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25571
    322
    https://api.yelp.com/v3/businesses/search?term=chinese&location=77336
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77336
    323
    https://api.yelp.com/v3/businesses/search?term=chinese&location=76556
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76556
    324
    https://api.yelp.com/v3/businesses/search?term=chinese&location=66088
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=66088
    325
    https://api.yelp.com/v3/businesses/search?term=chinese&location=23602
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=23602
    326
    https://api.yelp.com/v3/businesses/search?term=chinese&location=18914
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=18914
    327
    https://api.yelp.com/v3/businesses/search?term=chinese&location=4268
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=4268
    328
    https://api.yelp.com/v3/businesses/search?term=chinese&location=65017
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=65017
    329
    https://api.yelp.com/v3/businesses/search?term=chinese&location=92111
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92111
    330
    https://api.yelp.com/v3/businesses/search?term=chinese&location=92543
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92543
    331
    https://api.yelp.com/v3/businesses/search?term=chinese&location=27283
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27283
    332
    https://api.yelp.com/v3/businesses/search?term=chinese&location=18042
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=18042
    333
    https://api.yelp.com/v3/businesses/search?term=chinese&location=77807
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77807
    334
    https://api.yelp.com/v3/businesses/search?term=chinese&location=31405
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=31405
    335
    https://api.yelp.com/v3/businesses/search?term=chinese&location=59820
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=59820
    336
    https://api.yelp.com/v3/businesses/search?term=chinese&location=57105
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57105
    337
    https://api.yelp.com/v3/businesses/search?term=chinese&location=71834
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71834
    338
    https://api.yelp.com/v3/businesses/search?term=chinese&location=36558
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=36558
    339
    https://api.yelp.com/v3/businesses/search?term=chinese&location=47436
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47436
    340
    https://api.yelp.com/v3/businesses/search?term=chinese&location=11693
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=11693
    341
    https://api.yelp.com/v3/businesses/search?term=chinese&location=27317
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27317
    342
    https://api.yelp.com/v3/businesses/search?term=chinese&location=55976
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55976
    343
    https://api.yelp.com/v3/businesses/search?term=chinese&location=6422
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6422
    344
    https://api.yelp.com/v3/businesses/search?term=chinese&location=75454
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=75454
    345
    https://api.yelp.com/v3/businesses/search?term=chinese&location=47670
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47670
    346
    https://api.yelp.com/v3/businesses/search?term=chinese&location=77373
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77373
    347
    https://api.yelp.com/v3/businesses/search?term=chinese&location=17327
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17327
    348
    https://api.yelp.com/v3/businesses/search?term=chinese&location=23228
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=23228
    349
    https://api.yelp.com/v3/businesses/search?term=chinese&location=10303
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=10303
    350
    https://api.yelp.com/v3/businesses/search?term=chinese&location=20634
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=20634
    351
    https://api.yelp.com/v3/businesses/search?term=chinese&location=91207
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=91207
    352
    https://api.yelp.com/v3/businesses/search?term=chinese&location=61062
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=61062
    353
    https://api.yelp.com/v3/businesses/search?term=chinese&location=17774
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17774
    354
    https://api.yelp.com/v3/businesses/search?term=chinese&location=91405
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=91405
    355
    https://api.yelp.com/v3/businesses/search?term=chinese&location=2717
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=2717
    356
    https://api.yelp.com/v3/businesses/search?term=chinese&location=13068
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13068
    357
    https://api.yelp.com/v3/businesses/search?term=chinese&location=71957
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71957
    358
    https://api.yelp.com/v3/businesses/search?term=chinese&location=65332
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=65332
    359
    https://api.yelp.com/v3/businesses/search?term=chinese&location=97366
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97366
    360
    https://api.yelp.com/v3/businesses/search?term=chinese&location=54747
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54747
    361
    https://api.yelp.com/v3/businesses/search?term=chinese&location=79703
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=79703
    362
    https://api.yelp.com/v3/businesses/search?term=chinese&location=30666
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30666
    363
    https://api.yelp.com/v3/businesses/search?term=chinese&location=88337
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=88337
    364
    https://api.yelp.com/v3/businesses/search?term=chinese&location=63304
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=63304
    365
    https://api.yelp.com/v3/businesses/search?term=chinese&location=33496
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33496
    366
    https://api.yelp.com/v3/businesses/search?term=chinese&location=15656
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=15656
    367
    https://api.yelp.com/v3/businesses/search?term=chinese&location=71340
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71340
    368
    https://api.yelp.com/v3/businesses/search?term=chinese&location=35475
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35475
    369
    https://api.yelp.com/v3/businesses/search?term=chinese&location=64081
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=64081
    370
    https://api.yelp.com/v3/businesses/search?term=chinese&location=43804
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=43804
    371
    https://api.yelp.com/v3/businesses/search?term=chinese&location=3048
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=3048
    372
    https://api.yelp.com/v3/businesses/search?term=chinese&location=45780
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=45780
    373
    https://api.yelp.com/v3/businesses/search?term=chinese&location=6801
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6801
    374
    https://api.yelp.com/v3/businesses/search?term=chinese&location=14131
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14131
    375
    https://api.yelp.com/v3/businesses/search?term=chinese&location=92310
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=92310
    376
    https://api.yelp.com/v3/businesses/search?term=chinese&location=49036
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49036
    377
    https://api.yelp.com/v3/businesses/search?term=chinese&location=52320
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=52320
    378
    https://api.yelp.com/v3/businesses/search?term=chinese&location=54229
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54229
    379
    https://api.yelp.com/v3/businesses/search?term=chinese&location=12010
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12010
    380
    https://api.yelp.com/v3/businesses/search?term=chinese&location=76943
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76943
    381
    https://api.yelp.com/v3/businesses/search?term=chinese&location=39180
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=39180
    382
    https://api.yelp.com/v3/businesses/search?term=chinese&location=46524
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=46524
    383
    https://api.yelp.com/v3/businesses/search?term=chinese&location=33559
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33559
    384
    https://api.yelp.com/v3/businesses/search?term=chinese&location=7606
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=7606
    385
    https://api.yelp.com/v3/businesses/search?term=chinese&location=28635
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28635
    386
    https://api.yelp.com/v3/businesses/search?term=chinese&location=12723
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12723
    387
    https://api.yelp.com/v3/businesses/search?term=chinese&location=12804
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12804
    388
    https://api.yelp.com/v3/businesses/search?term=chinese&location=53018
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=53018
    389
    https://api.yelp.com/v3/businesses/search?term=chinese&location=13335
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13335
    390
    https://api.yelp.com/v3/businesses/search?term=chinese&location=27288
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27288
    391
    https://api.yelp.com/v3/businesses/search?term=chinese&location=95237
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=95237
    392
    https://api.yelp.com/v3/businesses/search?term=chinese&location=17018
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17018
    393
    https://api.yelp.com/v3/businesses/search?term=chinese&location=17980
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17980
    394
    https://api.yelp.com/v3/businesses/search?term=chinese&location=22642
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22642
    395
    https://api.yelp.com/v3/businesses/search?term=chinese&location=67839
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=67839
    396
    https://api.yelp.com/v3/businesses/search?term=chinese&location=5454
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=5454
    397
    https://api.yelp.com/v3/businesses/search?term=chinese&location=93925
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=93925
    398
    https://api.yelp.com/v3/businesses/search?term=chinese&location=12941
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12941
    399
    https://api.yelp.com/v3/businesses/search?term=chinese&location=83849
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=83849
    400
    https://api.yelp.com/v3/businesses/search?term=chinese&location=61606
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=61606
    401
    https://api.yelp.com/v3/businesses/search?term=chinese&location=57315
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57315
    402
    https://api.yelp.com/v3/businesses/search?term=chinese&location=94040
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=94040
    403
    https://api.yelp.com/v3/businesses/search?term=chinese&location=26205
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=26205
    404
    https://api.yelp.com/v3/businesses/search?term=chinese&location=13326
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=13326
    405
    https://api.yelp.com/v3/businesses/search?term=chinese&location=85553
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85553
    406
    https://api.yelp.com/v3/businesses/search?term=chinese&location=59802
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=59802
    407
    https://api.yelp.com/v3/businesses/search?term=chinese&location=1922
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=1922
    408
    https://api.yelp.com/v3/businesses/search?term=chinese&location=8854
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=8854
    409
    https://api.yelp.com/v3/businesses/search?term=chinese&location=12831
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12831
    410
    https://api.yelp.com/v3/businesses/search?term=chinese&location=7603
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=7603
    411
    https://api.yelp.com/v3/businesses/search?term=chinese&location=8540
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=8540
    412
    https://api.yelp.com/v3/businesses/search?term=chinese&location=45005
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=45005
    413
    https://api.yelp.com/v3/businesses/search?term=chinese&location=14033
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=14033
    414
    https://api.yelp.com/v3/businesses/search?term=chinese&location=43758
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=43758
    415
    https://api.yelp.com/v3/businesses/search?term=chinese&location=76525
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76525
    416
    https://api.yelp.com/v3/businesses/search?term=chinese&location=45684
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=45684
    417
    https://api.yelp.com/v3/businesses/search?term=chinese&location=33025
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33025
    418
    https://api.yelp.com/v3/businesses/search?term=chinese&location=25427
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25427
    419
    https://api.yelp.com/v3/businesses/search?term=chinese&location=72007
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72007
    420
    https://api.yelp.com/v3/businesses/search?term=chinese&location=48386
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48386
    421
    https://api.yelp.com/v3/businesses/search?term=chinese&location=42025
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=42025
    422
    https://api.yelp.com/v3/businesses/search?term=chinese&location=10553
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=10553
    423
    https://api.yelp.com/v3/businesses/search?term=chinese&location=6516
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6516
    424
    https://api.yelp.com/v3/businesses/search?term=chinese&location=6069
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6069
    425
    https://api.yelp.com/v3/businesses/search?term=chinese&location=46617
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=46617
    426
    https://api.yelp.com/v3/businesses/search?term=chinese&location=1430
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=1430
    427
    https://api.yelp.com/v3/businesses/search?term=chinese&location=70748
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=70748
    428
    https://api.yelp.com/v3/businesses/search?term=chinese&location=49893
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49893
    429
    https://api.yelp.com/v3/businesses/search?term=chinese&location=63935
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=63935
    430
    https://api.yelp.com/v3/businesses/search?term=chinese&location=25401
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25401
    431
    https://api.yelp.com/v3/businesses/search?term=chinese&location=41564
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=41564
    432
    https://api.yelp.com/v3/businesses/search?term=chinese&location=48851
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=48851
    433
    https://api.yelp.com/v3/businesses/search?term=chinese&location=58647
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=58647
    434
    https://api.yelp.com/v3/businesses/search?term=chinese&location=27549
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27549
    435
    https://api.yelp.com/v3/businesses/search?term=chinese&location=4107
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=4107
    436
    https://api.yelp.com/v3/businesses/search?term=chinese&location=55419
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55419
    437
    https://api.yelp.com/v3/businesses/search?term=chinese&location=19057
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=19057
    438
    https://api.yelp.com/v3/businesses/search?term=chinese&location=68147
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=68147
    439
    https://api.yelp.com/v3/businesses/search?term=chinese&location=25253
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=25253
    440
    https://api.yelp.com/v3/businesses/search?term=chinese&location=56726
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=56726
    Uh oh
    441
    https://api.yelp.com/v3/businesses/search?term=chinese&location=19076
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=19076
    442
    https://api.yelp.com/v3/businesses/search?term=chinese&location=49411
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49411
    443
    https://api.yelp.com/v3/businesses/search?term=chinese&location=46312
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=46312
    444
    https://api.yelp.com/v3/businesses/search?term=chinese&location=21061
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=21061
    445
    https://api.yelp.com/v3/businesses/search?term=chinese&location=95240
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=95240
    446
    https://api.yelp.com/v3/businesses/search?term=chinese&location=35801
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=35801
    447
    https://api.yelp.com/v3/businesses/search?term=chinese&location=30152
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=30152
    448
    https://api.yelp.com/v3/businesses/search?term=chinese&location=32444
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=32444
    449
    https://api.yelp.com/v3/businesses/search?term=chinese&location=79118
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=79118
    450
    https://api.yelp.com/v3/businesses/search?term=chinese&location=71031
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71031
    451
    https://api.yelp.com/v3/businesses/search?term=chinese&location=49234
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49234
    452
    https://api.yelp.com/v3/businesses/search?term=chinese&location=46323
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=46323
    453
    https://api.yelp.com/v3/businesses/search?term=chinese&location=55802
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55802
    454
    https://api.yelp.com/v3/businesses/search?term=chinese&location=85606
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85606
    455
    https://api.yelp.com/v3/businesses/search?term=chinese&location=27541
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27541
    456
    https://api.yelp.com/v3/businesses/search?term=chinese&location=11385
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=11385
    457
    https://api.yelp.com/v3/businesses/search?term=chinese&location=47591
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47591
    458
    https://api.yelp.com/v3/businesses/search?term=chinese&location=37764
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37764
    459
    https://api.yelp.com/v3/businesses/search?term=chinese&location=60305
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60305
    460
    https://api.yelp.com/v3/businesses/search?term=chinese&location=42717
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=42717
    461
    https://api.yelp.com/v3/businesses/search?term=chinese&location=98270
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98270
    462
    https://api.yelp.com/v3/businesses/search?term=chinese&location=40769
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=40769
    463
    https://api.yelp.com/v3/businesses/search?term=chinese&location=47003
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=47003
    464
    https://api.yelp.com/v3/businesses/search?term=chinese&location=68930
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=68930
    465
    https://api.yelp.com/v3/businesses/search?term=chinese&location=61270
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=61270
    466
    https://api.yelp.com/v3/businesses/search?term=chinese&location=32187
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=32187
    467
    https://api.yelp.com/v3/businesses/search?term=chinese&location=97321
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=97321
    468
    https://api.yelp.com/v3/businesses/search?term=chinese&location=93637
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=93637
    469
    https://api.yelp.com/v3/businesses/search?term=chinese&location=76706
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=76706
    470
    https://api.yelp.com/v3/businesses/search?term=chinese&location=50543
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=50543
    471
    https://api.yelp.com/v3/businesses/search?term=chinese&location=55069
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55069
    472
    https://api.yelp.com/v3/businesses/search?term=chinese&location=29113
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=29113
    473
    https://api.yelp.com/v3/businesses/search?term=chinese&location=98394
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=98394
    474
    https://api.yelp.com/v3/businesses/search?term=chinese&location=22952
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22952
    475
    https://api.yelp.com/v3/businesses/search?term=chinese&location=18237
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=18237
    476
    https://api.yelp.com/v3/businesses/search?term=chinese&location=72833
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=72833
    477
    https://api.yelp.com/v3/businesses/search?term=chinese&location=55060
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=55060
    478
    https://api.yelp.com/v3/businesses/search?term=chinese&location=60468
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60468
    479
    https://api.yelp.com/v3/businesses/search?term=chinese&location=54979
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=54979
    480
    https://api.yelp.com/v3/businesses/search?term=chinese&location=17051
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17051
    481
    https://api.yelp.com/v3/businesses/search?term=chinese&location=4622
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=4622
    482
    https://api.yelp.com/v3/businesses/search?term=chinese&location=85353
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=85353
    483
    https://api.yelp.com/v3/businesses/search?term=chinese&location=10984
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=10984
    484
    https://api.yelp.com/v3/businesses/search?term=chinese&location=60433
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60433
    485
    https://api.yelp.com/v3/businesses/search?term=chinese&location=42366
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=42366
    486
    https://api.yelp.com/v3/businesses/search?term=chinese&location=22027
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22027
    487
    https://api.yelp.com/v3/businesses/search?term=chinese&location=17214
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=17214
    488
    https://api.yelp.com/v3/businesses/search?term=chinese&location=79705
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=79705
    489
    https://api.yelp.com/v3/businesses/search?term=chinese&location=71046
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=71046
    490
    https://api.yelp.com/v3/businesses/search?term=chinese&location=12759
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12759
    491
    https://api.yelp.com/v3/businesses/search?term=chinese&location=37101
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=37101
    492
    https://api.yelp.com/v3/businesses/search?term=chinese&location=28226
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28226
    493
    https://api.yelp.com/v3/businesses/search?term=chinese&location=23219
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=23219
    494
    https://api.yelp.com/v3/businesses/search?term=chinese&location=624
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=624
    495
    https://api.yelp.com/v3/businesses/search?term=chinese&location=80831
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=80831
    496
    https://api.yelp.com/v3/businesses/search?term=chinese&location=28782
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=28782
    497
    https://api.yelp.com/v3/businesses/search?term=chinese&location=78941
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=78941
    498
    https://api.yelp.com/v3/businesses/search?term=chinese&location=32025
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=32025
    499
    https://api.yelp.com/v3/businesses/search?term=chinese&location=61604
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=61604
    500
    https://api.yelp.com/v3/businesses/search?term=chinese&location=5261
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=5261
    501
    https://api.yelp.com/v3/businesses/search?term=chinese&location=59522
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=59522
    Uh oh
    502
    https://api.yelp.com/v3/businesses/search?term=chinese&location=22191
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=22191
    503
    https://api.yelp.com/v3/businesses/search?term=chinese&location=6377
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=6377
    504
    https://api.yelp.com/v3/businesses/search?term=chinese&location=27043
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=27043
    505
    https://api.yelp.com/v3/businesses/search?term=chinese&location=60018
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=60018
    506
    https://api.yelp.com/v3/businesses/search?term=chinese&location=33542
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=33542
    507
    https://api.yelp.com/v3/businesses/search?term=chinese&location=80482
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=80482
    508
    https://api.yelp.com/v3/businesses/search?term=chinese&location=93545
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=93545
    509
    https://api.yelp.com/v3/businesses/search?term=chinese&location=75149
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=75149
    510
    https://api.yelp.com/v3/businesses/search?term=chinese&location=79764
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=79764
    511
    https://api.yelp.com/v3/businesses/search?term=chinese&location=41073
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=41073
    512
    https://api.yelp.com/v3/businesses/search?term=chinese&location=53576
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=53576
    513
    https://api.yelp.com/v3/businesses/search?term=chinese&location=16153
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=16153
    514
    https://api.yelp.com/v3/businesses/search?term=chinese&location=12148
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=12148
    515
    https://api.yelp.com/v3/businesses/search?term=chinese&location=67835
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=67835
    516
    https://api.yelp.com/v3/businesses/search?term=chinese&location=77520
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=77520
    517
    https://api.yelp.com/v3/businesses/search?term=chinese&location=62613
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=62613
    518
    https://api.yelp.com/v3/businesses/search?term=chinese&location=57369
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=57369
    519
    https://api.yelp.com/v3/businesses/search?term=chinese&location=16870
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=16870
    520
    https://api.yelp.com/v3/businesses/search?term=chinese&location=49048
    https://api.yelp.com/v3/businesses/search?term=Mexican&location=49048
    


```python
# Preview chinese Data
chinese_data.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Zip Code</th>
      <th>chinese Review Count</th>
      <th>chinese Average Rating</th>
      <th>chinese Weighted Rating</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>74</th>
      <td>31771</td>
      <td>62</td>
      <td>3.3871</td>
      <td>210</td>
    </tr>
    <tr>
      <th>398</th>
      <td>47952</td>
      <td>54</td>
      <td>3.44444</td>
      <td>186</td>
    </tr>
    <tr>
      <th>579</th>
      <td>74956</td>
      <td>92</td>
      <td>3.27717</td>
      <td>301.5</td>
    </tr>
    <tr>
      <th>590</th>
      <td>33129</td>
      <td>2755</td>
      <td>3.47623</td>
      <td>9577</td>
    </tr>
    <tr>
      <th>562</th>
      <td>16930</td>
      <td>20</td>
      <td>3.8</td>
      <td>76</td>
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
      <th>74</th>
      <td>31771</td>
      <td>157</td>
      <td>3.7293</td>
      <td>585.5</td>
    </tr>
    <tr>
      <th>398</th>
      <td>47952</td>
      <td>183</td>
      <td>3.88525</td>
      <td>711</td>
    </tr>
    <tr>
      <th>579</th>
      <td>74956</td>
      <td>304</td>
      <td>4.00822</td>
      <td>1218.5</td>
    </tr>
    <tr>
      <th>590</th>
      <td>33129</td>
      <td>3022</td>
      <td>3.67852</td>
      <td>11116.5</td>
    </tr>
    <tr>
      <th>562</th>
      <td>16930</td>
      <td>421</td>
      <td>3.48812</td>
      <td>1468.5</td>
    </tr>
  </tbody>
</table>
</div>



## Calculate Summaries


```python
# Total Mexican Reviews
mexican_data["Mexican Review Count"].sum()
```




    477548




```python
# Total chinese Reviews
chinese_data["chinese Review Count"].sum()
```




    347549




```python
# Average Mexican Rating
mexican_data["Mexican Weighted Rating"].sum() / mexican_data["Mexican Review Count"].sum()
```




    3.9091839982577667




```python
# Average chinese Rating
chinese_data["chinese Weighted Rating"].sum() / chinese_data["chinese Review Count"].sum()
```




    3.753521086235322




```python
# Combine DataFrames into a single DataFrame
combined_data = pd.merge(mexican_data, chinese_data, on="Zip Code")
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
      <th>chinese Review Count</th>
      <th>chinese Average Rating</th>
      <th>chinese Weighted Rating</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>31771</td>
      <td>157</td>
      <td>3.7293</td>
      <td>585.5</td>
      <td>62</td>
      <td>3.3871</td>
      <td>210</td>
    </tr>
    <tr>
      <th>1</th>
      <td>47952</td>
      <td>183</td>
      <td>3.88525</td>
      <td>711</td>
      <td>54</td>
      <td>3.44444</td>
      <td>186</td>
    </tr>
    <tr>
      <th>2</th>
      <td>74956</td>
      <td>304</td>
      <td>4.00822</td>
      <td>1218.5</td>
      <td>92</td>
      <td>3.27717</td>
      <td>301.5</td>
    </tr>
    <tr>
      <th>3</th>
      <td>33129</td>
      <td>3022</td>
      <td>3.67852</td>
      <td>11116.5</td>
      <td>2755</td>
      <td>3.47623</td>
      <td>9577</td>
    </tr>
    <tr>
      <th>4</th>
      <td>16930</td>
      <td>421</td>
      <td>3.48812</td>
      <td>1468.5</td>
      <td>20</td>
      <td>3.8</td>
      <td>76</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Determine Total Review Count and Rating "Wins" by City (Winner Take All)
combined_data["Rating Wins"] = np.where(combined_data["Mexican Average Rating"] > combined_data["chinese Average Rating"], "Mexican", "chinese")
combined_data["Review Count Wins"] = np.where(combined_data["Mexican Review Count"] > combined_data["chinese Review Count"], "Mexican", "chinese")
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
      <th>chinese Review Count</th>
      <th>chinese Average Rating</th>
      <th>chinese Weighted Rating</th>
      <th>Rating Wins</th>
      <th>Review Count Wins</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>31771</td>
      <td>157</td>
      <td>3.7293</td>
      <td>585.5</td>
      <td>62</td>
      <td>3.3871</td>
      <td>210</td>
      <td>Mexican</td>
      <td>Mexican</td>
    </tr>
    <tr>
      <th>1</th>
      <td>47952</td>
      <td>183</td>
      <td>3.88525</td>
      <td>711</td>
      <td>54</td>
      <td>3.44444</td>
      <td>186</td>
      <td>Mexican</td>
      <td>Mexican</td>
    </tr>
    <tr>
      <th>2</th>
      <td>74956</td>
      <td>304</td>
      <td>4.00822</td>
      <td>1218.5</td>
      <td>92</td>
      <td>3.27717</td>
      <td>301.5</td>
      <td>Mexican</td>
      <td>Mexican</td>
    </tr>
    <tr>
      <th>3</th>
      <td>33129</td>
      <td>3022</td>
      <td>3.67852</td>
      <td>11116.5</td>
      <td>2755</td>
      <td>3.47623</td>
      <td>9577</td>
      <td>Mexican</td>
      <td>Mexican</td>
    </tr>
    <tr>
      <th>4</th>
      <td>16930</td>
      <td>421</td>
      <td>3.48812</td>
      <td>1468.5</td>
      <td>20</td>
      <td>3.8</td>
      <td>76</td>
      <td>chinese</td>
      <td>Mexican</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Tally number of cities where one food variety wins on ratings over the other
combined_data["Rating Wins"].value_counts()
```




    Mexican    364
    chinese    153
    Name: Rating Wins, dtype: int64




```python
# Tally number of cities where one food variety wins on review counts over the other
combined_data["Review Count Wins"].value_counts()
```




    Mexican    396
    chinese    121
    Name: Review Count Wins, dtype: int64



## Display Summary of Results


```python
# Model 1: Head-to-Head Review Counts
chinese_summary = pd.DataFrame({"Review Counts": chinese_data["chinese Review Count"].sum(),
                                "Rating Average": chinese_data["chinese Average Rating"].mean(),
                                "Review Count Wins": combined_data["Review Count Wins"].value_counts()["chinese"],
                                "Rating Wins": combined_data["Rating Wins"].value_counts()["chinese"]}, index=["chinese"])

mexican_summary = pd.DataFrame({"Review Counts": mexican_data["Mexican Review Count"].sum(),
                                "Rating Average": mexican_data["Mexican Average Rating"].mean(),
                                "Review Count Wins": combined_data["Review Count Wins"].value_counts()["Mexican"],
                                "Rating Wins": combined_data["Rating Wins"].value_counts()["Mexican"]}, index=["Mexican"])

final_summary = pd.concat([mexican_summary, chinese_summary])
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
      <td>3.827798</td>
      <td>364</td>
      <td>396</td>
      <td>477548</td>
    </tr>
    <tr>
      <th>chinese</th>
      <td>3.652124</td>
      <td>153</td>
      <td>121</td>
      <td>347549</td>
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
plt.ylabel("Review Counts")
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
# Histogram chinese Food (Ratings)
plt.figure()

# Subplot 1 (chinese)
plt.subplot(121)
combined_data["chinese Average Rating"].plot.hist(bins=[0, 0.5, 1, 1.5, 2, 2.5, 3, 3.5, 4, 4.5, 5.0], color="blue", alpha=0.6)
plt.xlabel("Chinese Restaurant Ratings")
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
chinese_ratings = combined_data["chinese Average Rating"]

mexican_review_counts = combined_data["Mexican Review Count"]
chinese_review_counts = combined_data["chinese Review Count"]
```


```python
# Run T-Test on Ratings
ttest_ind(mexican_ratings.values, chinese_ratings.values, equal_var=False)
```




    Ttest_indResult(statistic=nan, pvalue=nan)




```python
# Run T-Test on Review Counts
ttest_ind(mexican_review_counts.values, chinese_review_counts.values, equal_var=False)
```




    Ttest_indResult(statistic=3.1992212210289863, pvalue=0.0014203973434742572)



## Conclusions
---
Based on our analysis, it is clear that American preference for Mexican food far exceeds that of Chinese food. Both with regards to the  average consumer rating and the number of reviews given to such restaurants there is a clear statistical difference between Chinese and Mexican restaurants far in favor of the Mexican variety. 


```python

```

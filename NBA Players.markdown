# NBA: Grouping Players with K-Means

**Segmenting NBA players into groups with similar traits.**  

# 1. The Data 

We will be using this [dataset](http://www.databasebasketball.com/about/aboutstats.htm) of player performance from the 2013-2014 season.  

Below is our Data Dictionary:  

| Columns  | Definition |
| :------------ | :------------- |
| **player** | name of the player |
| **pos** | the position of the player |
| **g** | number of games the player was in|
| **pts** | total points the player scored |
| **fg.** | field goal percentage |
| **ft.** | free throw percentage |


```python
import numpy as np
import pandas as pd

nba = pd.read_csv("nba_2013.csv")
nba.head(3)
```




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>player</th>
      <th>pos</th>
      <th>age</th>
      <th>bref_team_id</th>
      <th>g</th>
      <th>gs</th>
      <th>mp</th>
      <th>fg</th>
      <th>fga</th>
      <th>fg.</th>
      <th>...</th>
      <th>drb</th>
      <th>trb</th>
      <th>ast</th>
      <th>stl</th>
      <th>blk</th>
      <th>tov</th>
      <th>pf</th>
      <th>pts</th>
      <th>season</th>
      <th>season_end</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Quincy Acy</td>
      <td>SF</td>
      <td>23</td>
      <td>TOT</td>
      <td>63</td>
      <td>0</td>
      <td>847</td>
      <td>66</td>
      <td>141</td>
      <td>0.468</td>
      <td>...</td>
      <td>144</td>
      <td>216</td>
      <td>28</td>
      <td>23</td>
      <td>26</td>
      <td>30</td>
      <td>122</td>
      <td>171</td>
      <td>2013-2014</td>
      <td>2013</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Steven Adams</td>
      <td>C</td>
      <td>20</td>
      <td>OKC</td>
      <td>81</td>
      <td>20</td>
      <td>1197</td>
      <td>93</td>
      <td>185</td>
      <td>0.503</td>
      <td>...</td>
      <td>190</td>
      <td>332</td>
      <td>43</td>
      <td>40</td>
      <td>57</td>
      <td>71</td>
      <td>203</td>
      <td>265</td>
      <td>2013-2014</td>
      <td>2013</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Jeff Adrien</td>
      <td>PF</td>
      <td>27</td>
      <td>TOT</td>
      <td>53</td>
      <td>12</td>
      <td>961</td>
      <td>143</td>
      <td>275</td>
      <td>0.520</td>
      <td>...</td>
      <td>204</td>
      <td>306</td>
      <td>38</td>
      <td>24</td>
      <td>36</td>
      <td>39</td>
      <td>108</td>
      <td>362</td>
      <td>2013-2014</td>
      <td>2013</td>
    </tr>
  </tbody>
</table>
<p>3 rows × 31 columns</p>
</div>



# 2. Point Guards

Point guards play one of the most crucial roles on a team because their primary responsibility is to create scoring opportunities for the team. We are going to focus our lesson on a machine learning technique called clustering, which allows us to visualize the types of point guards as well as group similar point guards together. For point guards, it's widely accepted that the `Assist to Turnover Ratio` is a good indicator for performance in games as it quantifies the number of scoring opportunities that player created. We will also use `Points Per Game`, since effective Point Guards not only set up scoring opportunities but also take a lot of the shots themselves.


```python
point_guards = nba[nba['pos'] == 'PG']
```

## 2.1 Points Per Game

Our dataset doesn't come with Points Per Game values, so we will need to calculate those values using each player's total points (`pts`) and the number of games (`g`) they played.


```python
point_guards['ppg'] = point_guards['pts'] / point_guards['g']

# Double check and make sure ppg = pts/g
point_guards[['pts', 'g', 'ppg']].head()
```

<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>pts</th>
      <th>g</th>
      <th>ppg</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>24</th>
      <td>930</td>
      <td>71</td>
      <td>13.098592</td>
    </tr>
    <tr>
      <th>29</th>
      <td>150</td>
      <td>20</td>
      <td>7.500000</td>
    </tr>
    <tr>
      <th>30</th>
      <td>660</td>
      <td>79</td>
      <td>8.354430</td>
    </tr>
    <tr>
      <th>38</th>
      <td>666</td>
      <td>72</td>
      <td>9.250000</td>
    </tr>
    <tr>
      <th>50</th>
      <td>378</td>
      <td>55</td>
      <td>6.872727</td>
    </tr>
  </tbody>
</table>
</div>



## 2.2 Assist Turnover Ration

We also need to create a column, `atr`, for the Assist Turnover Ratio, which is calculated by dividing total assists (`ast`) by total turnovers (`tov`):


```python
point_guards = point_guards[point_guards['tov'] != 0]
point_guards['atr'] = point_guards['ast'] / point_guards['tov']
```

## 2.3 Visualizing Point Guards

Using matplotlib we can create a scatter plot with `Points Per Game (ppg)` on the `X axis` and `Assist Turnover Ratio (atr)` on the `Y axis`.


```python
import matplotlib.pyplot as plt

plt.scatter(point_guards['ppg'], point_guards['atr'], c='y')
plt.title("Point Guards")
plt.xlabel('Points Per Game', fontsize=13)
plt.ylabel('Assist Turnover Ratio', fontsize=13)
plt.show()
```


![png](/assets/media/nba/NBA_10_0.png)


# 3. Clustering Players

There seem to be 5 general regions, or clusters, that the point guards fall into (with a few outliers). We will use a technique called clustering to segment all of the point guards into groups of alike players. While regression and other supervised machine learning techniques work well when we have a clear metric we want to optimize for and lots of pre-labelled data, we need to instead use unsupervised machine learning techniques to explore the structure within a data set that doesn't have a clear value to optimize.  

There are multiple ways of clustering data, here we will utilize centroid based clustering. Centroid based clustering works well when the clusters resemble circles with centers (or centroids). The centroid represent the arithmetic mean of all of the data points in that cluster.  

`K-Means` Clustering is a popular `centroid-based clustering` algorithm that we will use. The `K` in `K-Means` refers to the number of clusters we want to segment our data into. The key part with `K-Means` (and most unsupervised machine learning techniques) is that we have to specify what `k` is. There are advantages and disadvantages to this, but one advantage is that we can pick the `k` that makes the most sense for our use case. We will set `k` to `5` since we want `K-Means` to segment our data into `5 clusters`.

# 4. The Algorithm

`K-Means` is an iterative algorithm that switches between recalculating the centroid of each cluster and the players that belong to that cluster. To start, we will select 5 players at random and assign their coordinates as the initial centroids of the just created clusters.  

Our course of action is followed:  

`Step 1` (Assign Points to Clusters) For each player, we will calculate the Euclidean distance between that player's coordinates, or values for `atr` & `ppg`, and each of the centroids' coordinates. Assign the player to the cluster whose centroid is the closest to, or has the lowest Euclidean distance to, the player's values.  

`Step 2` (Update New Centroids of the Clusters) For each cluster, we will compute the new centroid by calculating the arithmetic mean of all of the points (players) in that cluster. We calculate the arithmetic mean by taking the average of all of the X values (`atr`) and the average of all of the Y values (`ppg`) of the points in that cluster.  

Finally we will iterate/repeat steps 1 & 2 until the clusters are no longer moving and have converged.


```python
num_clusters = 5
# Use numpy's random function to generate a list, length: num_clusters, of indices
random_initial_points = np.random.choice(point_guards.index, size=num_clusters)
# Use the random indices to create the centroids
centroids = point_guards.loc[random_initial_points]
```

## 4.1 Visualize Centroids

We will plot the `centroids`, in addition to the `point_guards`, so we can see where the randomly chosen centroids started out.


```python
plt.scatter(point_guards['ppg'], point_guards['atr'], c='yellow')
plt.scatter(centroids['ppg'], centroids['atr'], c='red')
plt.title("Centroids")
plt.xlabel('Points Per Game', fontsize=13)
plt.ylabel('Assist Turnover Ratio', fontsize=13)
plt.show()
```


![png](/assets/media/nba/NBA_15_0.png)


## 4.2 Assigning a Unique Identifier

While the `centroids` data frame object worked well for the initial centroids, where the centroids were just a subset of players, as we iterate the centroids' values will be coordinates that may not match another player's coordinates. Moving forward, we will use a dictionary object instead to represent the centroids.  

We need a unique identifier, like `cluster_id`, to refer to each cluster's centroid and a list representation of the centroid's coordinates (or values for `ppg` and `atr`). We will create a dictionary then with the following mapping:  

- key: `cluster_id` of that centroid's cluster
- value: centroid's coordinates expressed as a list ( `ppg` value first, `atr` value second )  

To generate the `cluster_ids`, we will iterate through each centroid and assign an integer from 0 to `k-1`. For example, the first centroid will have a `cluster_id` of 0, while the second one will have a `cluster_id` of 1. We will write a function, `centroids_to_dict`, that takes in the `centroids` data frame object, creates a `cluster_id` and converts the `ppg` and `atr` values for that centroid into a list of coordinates, and adds both the `cluster_id` and `coordinates_list` into the dictionary that's returned.


```python
def centroids_to_dict(centroids):
    dictionary = dict()
    # iterating counter we use to generate a cluster_id
    counter = 0

    # iterate a pandas data frame row-wise using .iterrows()
    for index, row in centroids.iterrows():
        coordinates = [row['ppg'], row['atr']]
        dictionary[counter] = coordinates
        counter += 1

    return dictionary

centroids_dict = centroids_to_dict(centroids)
```

# 5. Step 1: Euclidean Distance

Before we can assign players to clusters, we need a way to compare the `ppg` and `atr` values of the players with each cluster's centroids. `Euclidean distance` is the most common technique used in data science for measuring distance between vectors and works extremely well in 2 and 3 dimensions. While in higher dimensions, Euclidean distance can be misleading, in 2 dimensions Euclidean distance is essentially the Pythagorean theorem.  

We can create a function and call it `calculate_distance`, it will takes in 2 lists (the player's values for `ppg` and `atr` and the centroid's values for `ppg` and `atr`).


```python
import math

def calculate_distance(centroid, player_values):
    root_distance = 0
    
    for x in range(0, len(centroid)):
        difference = centroid[x] - player_values[x]
        squared_difference = difference**2
        root_distance += squared_difference

    euclid_distance = math.sqrt(root_distance)
    return euclid_distance

q = [5, 2]
p = [3,1]

# Sqrt(5) = ~2.24
print(calculate_distance(q, p))
```

    2.23606797749979


## 5.1 Assigning Data Points to Clusters

Now we need a way to assign data points to clusters based on Euclidean distance. Instead of creating a new variable or data structure to house the clusters, we will create a new column in our `point_guards` data frame that contains the `cluster_id` of the cluster it belongs to.


```python
# Add the function, `assign_to_cluster`
# This creates the column, `cluster`, by applying assign_to_cluster row-by-row
# Uncomment when ready

# point_guards['cluster'] = point_guards.apply(lambda row: assign_to_cluster(row), axis=1)
def assign_to_cluster(row):
    lowest_distance = -1
    closest_cluster = -1
    
    for cluster_id, centroid in centroids_dict.items():
        df_row = [row['ppg'], row['atr']]
        euclidean_distance = calculate_distance(centroid, df_row)
        
        if lowest_distance == -1:
            lowest_distance = euclidean_distance
            closest_cluster = cluster_id 
        elif euclidean_distance < lowest_distance:
            lowest_distance = euclidean_distance
            closest_cluster = cluster_id
    return closest_cluster

point_guards['cluster'] = point_guards.apply(lambda row: assign_to_cluster(row), axis=1)
```

## 5.2 Visualizing Clusters


```python
# Visualizing clusters
def visualize_clusters(df, num_clusters):
    colors = ['b', 'g', 'r', 'c', 'm', 'y', 'k']

    for n in range(num_clusters):
        clustered_df = df[df['cluster'] == n]
        plt.scatter(clustered_df['ppg'], clustered_df['atr'], c=colors[n-1])
        plt.xlabel('Points Per Game', fontsize=13)
        plt.ylabel('Assist Turnover Ratio', fontsize=13)
    plt.show()

visualize_clusters(point_guards, 5)
```


![png](/assets/media/nba/NBA_23_0.png)


# 6. Step 2: Recalculate Centroids


```python
def recalculate_centroids(df):
    new_centroids_dict = dict()
    # 0..1...2...3...4
    for cluster_id in range(0, num_clusters):
        # Finish the logic
        return new_centroids_dict

centroids_dict = recalculate_centroids(point_guards)
def recalculate_centroids(df):
    new_centroids_dict = dict()
    
    for cluster_id in range(0, num_clusters):
        values_in_cluster = df[df['cluster'] == cluster_id]
        # Calculate new centroid using mean of values in the cluster
        new_centroid = [np.average(values_in_cluster['ppg']), np.average(values_in_cluster['atr'])]
        new_centroids_dict[cluster_id] = new_centroid
    return new_centroids_dict

centroids_dict = recalculate_centroids(point_guards)
```

# 7. Repeat Step 1


```python
point_guards['cluster'] = point_guards.apply(lambda row: assign_to_cluster(row), axis=1)
visualize_clusters(point_guards, num_clusters)
```


![png](/assets/media/nba/NBA_27_0.png)


# 8. Repeat Step 1 and Step 2


```python
centroids_dict = recalculate_centroids(point_guards)
point_guards['cluster'] = point_guards.apply(lambda row: assign_to_cluster(row), axis=1)
visualize_clusters(point_guards, num_clusters)
```


![png](/assets/media/nba/NBA_29_0.png)


# 9. The Challenges of K-Means

As you repeat Steps 1 and 2 and run `visualize_clusters`, you will notice that a few of the points are changing clusters between every iteration (especially in areas where 2 clusters almost overlap), but otherwise, the clusters visually look like they don't move a lot after every iteration. This means 2 things:  

- K-Means doesn't cause massive changes in the makeup of clusters between iterations, meaning that it will always converge and become stable
- Because K-Means is conservative between iterations, where we pick the initial centroids and how we assign the players to clusters initially matters a lot

# 10. Overcoming the Challenges with Scikit-Learn

To counteract these problems, the `sklearn` implementation of `K-Means` does some intelligent things like re-running the entire clustering process lots of times with random initial centroids so the final results are a little less biased on one passthrough's initial centroids.


```python
from sklearn.cluster import KMeans

kmeans = KMeans(n_clusters=num_clusters)
kmeans.fit(point_guards[['ppg', 'atr']])
point_guards['cluster'] = kmeans.labels_

visualize_clusters(point_guards, num_clusters)
```


![png](/assets/media/nba/NBA_32_0.png)


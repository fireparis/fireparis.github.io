<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script> 

# Taxonomy of the Dow Jones Industrial Average [1/2]: the Minimum Spanning Tree case

In this short post, we analyse the correlations between the 30 constituents of the DJIA. The daily prices of all US stocks and ETF are available
on [Kaggle](https://www.kaggle.com/borismarjanovic/price-volume-data-for-all-us-stocks-etfs). This dataset is made of one file for each security containing the date, the open, high, low and close prices, the volume and the open interest. The approach followed is the following:
1. First, we load the datasets of all DJIA stocks and compute the daily returns
2. We then select the days for which the data for all 30 stocks are available
3. On this period, we compute the pairwise correlations and pairwise distances derived from these quantities
4. Finally, we compute the minimum spanning tree associated with these distances

## Data Loading

The file DOW.csv contains te tickers and names of the 30 companies composing the DJIA. The first step is to load this file in order to have access to the tickers list:
```python
import os
import pandas as pd
import numpy as np

constituents = pd.read_csv('Data/dow.csv', sep='\t', header=None)
tickers = constituents[0].apply(str).values
```
```tickers``` is an array containing the symbols of the 30 stocks.

Now, we have to load the price data of these specific stocks. We must pay attention to the empty files of the folder.
```python
os.chdir('Data/Stocks') # We move to the folder containing the txt stock files
filenames = [file for file in os.listdir() if file.endswith('.txt')\
             and os.path.getsize(file) > 0 and file.split('.')[0].upper() in tickers]
filenames = sorted(filenames)
```
The list ```filenames``` contains the names of the datasets of the 30 stocks. We then load all these files: 
```python
list_df = [] # This list will contain the 30 dataframes
for file in filenames:
    df = pd.read_csv(file)
    df = df[['Date', 'Close']] # Open, High, Low, Volume and OpenInt are not relevant in our case
    df.index = df.Date
    df['return'] = np.log(df.Close) - np.log(df['Close'].shift())
    df.drop(labels=['Date', 'Close'], inplace=True, axis=1)
    df.columns = [file.split('.')[0].upper()]
    list_df.append(df)
    
data  = pd.concat(list_df, axis=1) # The 30 dataframes are merged on their index (date)
data.dropna(axis=0, how='any', inplace=True) # We select only the period when all stocks are listed
```
This dataframe has 30 columns (one for each stock) and 2430 rows (about 10 years of daily returns). 

## Correlations

We can now compute the correlation matrix:
```python
corr = data.corr()
```
These correlation coefficients $$\rho_{ij}$$ can be turned into distances using the formula $$d_{ij} = \sqrt{2(1 - \rho_{ij})}$$. We can show that this function statisfies all the properties of a mathematical distance. So, we transform the correlation matrix into a distance matrix:
```python
def distance(rho):
    return 10 * np.sqrt(2 * (1 - rho))

dist_matrix = distance(corr).values
```
The prefactor in front of the distance is used to make future plots clearer.

## Minimum Spanning Tree

The matrix ```dist_matrix``` can be thought as the adjacency matrix of a particular undirected graph, where $$d_{ij}$$ is the distance associated with the edge connecting nodes $$i$$ and $$j$$. For instance, the stock nearest Apple (AAPL) is Intel (INTC). On this graph, we can define some trees which are defined as connex graphs without any cycle. In particular, we can consider the trees containing all the 30 stocks and define their length as the sum of the distances between each node. A minimum spanning tree is a tree of minimum length containing all the stocks. The image below shows a graph and the associated minimum spanning tree.

![Example of a minimum spanning tree](min_tree.png "Example of a minimum spanning tree")

There are three common algorithms to find this tree:
* Otakar Borůvka's algorithm
* Prim's algorithm
* Kruskal's algorithm

The complexity is $$\mathcal{O}(m\log{}n)$$ where $$m$$ is the number of edges and $$n$$ the number of vertices , except for Prim's algorithm where it can be $$\mathcal{O}(m + n\log{}n)$$ depending on the graph.

In python, the libraries ```networkx``` and ```scipy``` provide some tools to deal with graphs. In particular, ```scipy``` allows to compute the minimum spanning tree directly from the adjacency matrix: 
```python
from scipy.sparse.csgraph import minimum_spanning_tree

tree = minimum_spanning_tree(dist_matrix)
tree = tree.toarray().astype(float)
```
We can now use ```networkx``` to draw the minimum spanning tree found above:
```python
import networkx as nx
import matplotlib.pyplot as plt

labels = list(data.columns) # tickers
graph = nx.from_numpy_matrix(tree)
mapping = dict(zip(graph, labels))
graph = nx.relabel_nodes(graph, mapping) # nodes are characters a through z

plt.figure(figsize=(10,10))
plt.axis('off')
nx.draw_networkx(graph, c='r', alpha=0.7)
```

![DJIA minimum spanning tree](tree.png "DJIA minimum spanning tree")

Economic clusters are clearly observable on this tree: at the top-right corner, we can spot tech companies grouped together, while financial service companies are visible in the lower-left corner. Therefore, the classification obtained makes sense from an economic point of view.

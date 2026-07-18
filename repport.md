# Steam Game Similarity Graph — Community Detection

**Course:** Graph Algorithms  
**Instructor:** Dr. Avner Friel  
**Students:** Yanir Karpis, Chen Brown, Shaked Rosenberg  
**Date:** July 2026

## Abstract

This project studies whether groups of Steam games created from player behavior are similar to the games' content labels. We built a weighted graph where every node is a game. An edge connects two games when many players played both games. The edge weight is based on **Jaccard similarity**, so very popular games do not dominate the graph only because they have many players.

We used the **Louvain** algorithm to find communities in the graph. We also used **PageRank**, **Betweenness Centrality**, **Modularity**, and **Normalized Mutual Information (NMI)** to analyze and evaluate the results. Our final graph contains 333 games and 4,617 edges. Louvain found 18 communities with `modularity = 0.1723` and `NMI = 0.2973`. This NMI score is 1.4 times higher than a random baseline (`NMI = 0.2153`). Therefore, player behavior has a partial connection to Steam genre and tag labels, but the connection is not perfect. Very popular games connect players from different genres and create mixed communities.

## 1. Introduction and Motivation

Steam organizes games using genres and tags. These labels help users search for games and receive recommendations. However, official labels do not always show how players really choose and play games. For example, two games can have the same genre but attract different types of players. On the other hand, games from different genres can be played by many of the same users because they have a similar style, are popular, are cheap, belong to the same series, or are played with friends.

The goal of this project is to represent this behavior as a **game-similarity graph**. We then check whether the communities found in the graph are similar to Steam's content labels. This can help us understand player groups, improve recommendation systems, and study relationships between games.

Our main research question is:

> Do communities found from shared player behavior match Steam genre and tag labels?

This project does not try to predict the genre of one game. Instead, it checks whether an unsupervised graph algorithm creates groups that are similar to an outside classification of the same games.

## 2. Data and Preprocessing

The player behavior data comes from `steam-200k.csv`. Each row contains `user_id`, `game_title`, `behavior`, hours played, and an extra field. We kept only rows where `behavior = play`, because a purchase record alone does not prove that the user played the game. If a user had more than one record for the same game, we kept the record with the highest number of hours.

After this step, the dataset contained:

| Measure | Value |
|---|---:|
| `play` rows | 70,477 |
| Unique users | 11,350 |
| Unique games | 3,600 |

We also used `games.json`, which includes game names, genres, and community tags. We selected one main label for each game. First, we chose the highest-voted tag that was not a **META_TAG**. For example, labels such as `Singleplayer`, `Multiplayer`, `2D`, `Indie`, and `VR` were not used as main genre labels because they do not describe the game type clearly enough. If no useful tag was found, we used the first official genre.

Some detailed tags were merged into larger groups. For example, `FPS` and `Hero Shooter` were changed to `Shooter`, while `RTS` and `City Builder` were changed to `Strategy`.

Game names in the two datasets were not always written in the same way. We matched them in three steps. First, we used exact matching after converting names to lowercase, removing punctuation, and fixing spaces. Second, we used fuzzy matching with a score of at least 90 and a length-ratio check of 0.4. Finally, we used official genres as a fallback. The results were:

| Matching step | Matched games | Unmatched games |
|---|---:|---:|
| Exact matching | 2,509 | 1,091 |
| Fuzzy matching | 2,926 | 674 |
| Genre fallback | 3,007 | 593 |

In total, we matched 3,007 out of 3,600 games, which is 83.5%. The labels are not always official Steam genres. In most cases, they are cleaned and merged community tags. Therefore, our evaluation compares communities with the content labels created by our pipeline, not with one perfect ground-truth genre list.

## 3. Method

### 3.1 Building the Graph

We built an undirected graph `G = (V, E)`. Every node is a game with a known label. For each user, we collected all games that the user played. Then, for every pair of games played by the same user, we counted one shared player.

We added an edge only when at least 25 users played both games:

`shared(i, j) ≥ 25`

This threshold reduces weak and random connections. The edge weight is **Jaccard similarity**:

`w(i, j) = |P_i ∩ P_j| / |P_i ∪ P_j|`

Here, `P_i` is the set of players of game `i`. Jaccard similarity is important because raw shared-player counts favor very popular games. For example, a game with thousands of players may overlap with many other games just because it is popular. Jaccard measures the shared players compared with the total number of players in both games.

We kept only the **largest connected component** of the graph. The final graph was:

| Graph measure | Value |
|---|---:|
| Co-play pairs before threshold | 636,790 |
| Nodes in largest component | 333 |
| Edges | 4,617 |
| Average degree | 27.7 |

### 3.2 Louvain Community Detection

Louvain divides the graph into communities by trying to maximize weighted modularity. In simple words, it tries to find groups with many internal connections compared with the rest of the graph.

We used `weight='weight'`, `random_state=42`, and `resolution=1.5`. The random state makes the result repeatable. The resolution controls how detailed the result is. A higher resolution usually gives more, smaller communities.

We compared two resolution values. With `resolution=1.0`, Louvain found only 6 communities, including one large community with 145 games. With `resolution=1.5`, it found 18 communities. We chose 1.5 because it gave more useful and detailed groups, even though the modularity value at 1.0 was higher.

### 3.3 Centrality and Algorithm Comparison

We calculated **PageRank** using the edge weights. PageRank finds games that are connected to other important games. We also calculated **Betweenness Centrality** without edge weights. In NetworkX, an edge weight is treated as a distance. Since our weight is a similarity score, using it as a distance would give the opposite meaning.

We also tested **Girvan–Newman**. This method repeatedly removes edges with high edge betweenness. It was not suitable for our dense graph. Its best result after 30 splits was only two communities: 332 games and 1 game, with `modularity ≈ 0`. A graph with fewer edges, created with a higher shared-player threshold, may work better with this algorithm, but it would remove many smaller games.

## 4. Results and Findings

### 4.1 Community Quality and Label Agreement

The main results are shown below:

| Method / setting | Number of communities | Modularity | NMI |
|---|---:|---:|---:|
| Random baseline (100 permutations) | — | — | 0.2153 |
| Louvain, `resolution=1.5` | 18 | 0.1723 | 0.2973 |
| Louvain, `resolution=1.0` | 6 | 0.2100 | 0.1861 |

With `resolution=1.5`, the NMI score is 0.2973. It is 1.4 times higher than the random baseline. This means there is a real connection between player communities and content labels, but the connection is moderate rather than perfect.

The random baseline is not zero because NMI is affected by the number and size of groups and by the distribution of labels. For this reason, comparing the result with shuffled community labels is more useful than looking at the NMI score alone.

At `resolution=1.0`, modularity is higher (`0.2100`), but NMI is lower (`0.1861`) and there are only 6 communities. This shows that modularity and NMI measure different things. A result can have stronger graph structure while matching the external labels less well. Using both measures gives a better evaluation.

### 4.2 Community Examples

Some communities have a clear content identity. Community 3 has 8 `Call of Duty` games and is mostly labeled `Action`. Community 0 has 39 games and is mainly `Strategy`, including `Sid Meier's Civilization V` and `Total War SHOGUN 2`. Community 2 is mostly `Shooter` and includes `Half-Life 2` and `Counter-Strike Source`.

However, not every community matches one label. Community 4 is the largest community with 92 games. It includes games labeled `Action`, `Shooter`, `RPG`, `Survival`, and `Sandbox`. Important games in this community are `Team Fortress 2`, `Garry's Mod`, `Counter-Strike Global Offensive`, `Unturned`, and `Warframe`.

This mixed community is not only an algorithm mistake. It shows that popular games can connect many different types of players. These games act as common meeting points between several genres.

### 4.3 Hub and Bridge Games

The highest PageRank games were `The Elder Scrolls V Skyrim` (0.026751), `Left 4 Dead 2` (0.023315), `Team Fortress 2` (0.022090), `Portal 2` (0.021075), and `Borderlands 2` (0.020940). These games are connected to many other important games.

The highest Betweenness Centrality games were `Team Fortress 2` (0.285045), `The Elder Scrolls V Skyrim` (0.122195), `Dota 2` (0.107741), `Left 4 Dead 2` (0.083276), and `Counter-Strike Global Offensive` (0.078470).

`Team Fortress 2` is especially important because it is not only popular; it is also a bridge between different parts of the graph. This helps explain why NMI is only partial. Hub games connect communities that do not always have the same genre label.

### 4.4 Answer to the Research Question

The answer is **partial agreement**. Communities based on player behavior are not the same as Steam labels, but they are related to them more than random groups are. Games from the same series or with similar competitive and gameplay styles often appear together. At the same time, popular and multi-genre games create mixed communities and links between labels.

Therefore, player behavior includes information that is related to genre, but it also includes other factors that are not shown by Steam's content taxonomy.

## 5. Limitations

First, `steam-200k.csv` is a limited snapshot of users and games. It may not represent all Steam users or current Steam activity. Second, fuzzy name matching can create some wrong matches, even with a high score threshold. Third, using only one label for each game is a simplification because many games belong to more than one genre.

The shared-player threshold of 25 and the decision to keep only the largest connected component also affect the result. These choices reduce noise, but they remove smaller or niche games from the analysis. In addition, we selected `resolution=1.5` after reviewing the results. In future work, we should test a wider range of thresholds, resolutions, and random seeds.

Finally, NMI measures agreement between two partitions. It does not directly measure whether the communities would produce better game recommendations. A moderate NMI does not mean the graph failed. It may mean that player behavior contains useful information beyond genre labels.

## 6. Conclusion and Future Work

This project presents a full graph-analysis pipeline for Steam games. We cleaned user–game data, matched game labels, built a weighted Jaccard graph, used Louvain community detection, calculated centrality, and evaluated the communities with NMI.

The main result is that Louvain found 18 communities that have a partial but meaningful relationship with Steam content labels. The resolution comparison and the unsuccessful Girvan–Newman experiment show that choosing the right algorithm and parameters is important, and that one metric alone is not enough.

Possible future work includes:

- Testing more values of `MIN_SHARED_PLAYERS`.
- Comparing Louvain with Leiden and Label Propagation.
- Using multi-label evaluation instead of one label per game.
- Adding playing hours to the edge weight in a careful way.
- Testing community stability with different random seeds and data samples.
- Using the communities as a feature in a game recommendation system.

## 7. Individual Contributions

# 7. Individual Contributions

The core project workload was divided equally among team members, with each student taking ownership of one major programming phase, one analytical focus, and one reproducibility duty.

* **Yanir Karpis (Graph & Community Detection)**
    * **Code:** Built the NetworkX co-play graph pipeline, computed Jaccard similarity weights, and isolated the largest connected component.
    * **Analysis:** Implemented and tuned the Louvain community detection algorithm (comparing resolution 1.0 vs. 1.5) and tested the Girvan-Newman baseline.
    * **Reproducibility:** Secured code determinism with random state seeding and optimized execution runtimes in the Notebook.

* **Chen Brown (Data Engineering & Alignment)**
    * **Code:** Developed the pandas preprocessing pipeline, filtering for active play records and resolving duplicate user sessions.
    * **Analysis:** Structured the taxonomy mapping logic and implemented the multi-step string matching system (`thefuzz`) to align game titles.
    * **Reproducibility:** Configured the repository architecture, managed dependencies (`requirements.txt`), and automated data ingestion.

* **Shaked Rosenberg (Metrics, Evaluation & Synthesis)**
    * **Code:** Coded the topological network calculations, computing PageRank and Betweenness Centrality to identify network hubs and bridges.
    * **Analysis:** Created the statistical evaluation pipeline, calculating Normalized Mutual Information (NMI) against a 100-permutation random baseline.
    * **Reproducibility:** Structured the final technical report, interpreted the algorithmic results, and synthesized the domain findings.

## References

1. Blondel, V. D., Guillaume, J.-L., Lambiotte, R., & Lefebvre, E. (2008). *Fast unfolding of communities in large networks*. Journal of Statistical Mechanics: Theory and Experiment, P10008. https://doi.org/10.1088/1742-5468/2008/10/P10008
2. Newman, M. E. J., & Girvan, M. (2004). *Finding and evaluating community structure in networks*. Physical Review E, 69, 026113. https://doi.org/10.1103/PhysRevE.69.026113
3. Strehl, A., & Ghosh, J. (2002). *Cluster Ensembles — A Knowledge Reuse Framework for Combining Multiple Partitions*. Journal of Machine Learning Research, 3, 583–617. https://jmlr.csail.mit.edu/papers/v3/strehl02a.html
4. `steam-200k.csv` and `games.json`, the datasets used by the project Notebook and downloaded through its Google Drive code. https://www.kaggle.com/datasets/fronkongames/steam-games-dataset?resource=download

---

### Appendix: Reproducing the Analysis

The file `networks_algorithms_project_new1.ipynb` contains all analysis steps. Dependencies are listed in `requirements.txt`, including `networkx`, `python-louvain`, `pandas`, `scikit-learn`, `thefuzz`, `numpy`, `matplotlib`, `tqdm`, and `gdown`. A full rerun also requires access to the Google Drive folder defined in the Notebook.

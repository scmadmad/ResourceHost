## Offline Challenge
Author: Andong Ma
Date: 2020-04-15
## Problem
> Suppose we want to design a Leaderboards service that can hold up to 10 million rows per leaderboard and can handle high concurrency of requests. LeaderboardRow = {rank, userId, score}.
> 
 > Describe what data structure(s) you will use to hold the leaderboard in memory, optimizing for the following 3 APIs:
 > - void updateByUserId(userId, newScore)
 > - LeaderboardRow fetchByUserId(userId)
 > - LeaderboardRow fetchByRank(targetRank) 
 >
>Analyze and justify why your solution is a winner! What other solutions did you try but reject and why? Are there any assumptions or limitations that you need to call out?

## My Solution
For the Leaderboard service, if the user number is not huge, the implementation of the required API would be quite easy. However, in the given problem we have 10 million users, thus we need to use an advanced data structure to perform proper leaderboard calculations in a fast and cheap way.

My solution is to use Skip List + HashMap:
 - Skip List: to maintain the ranking of LeaderBoardRow
 - HashMap: to keep the mapping of userId -> LeaderBoardRow

#### What is Skip List?
Skip List is a probabilisitc data structure that is built upon the general idea of a linked list. The skip list uses probability to build subsequent layers of linked lists upon an original linked list. Each additional layer of links contains fewer elements, but no new elements. You can think about the skip list like a subway system. There's one train that stops at every single stop. However, there is also an express train. This train doesn't visit any unique stops, but it will stop at fewer stops. This makes the express train an attractive option if you know where it stops.

#### Why Skip List?
- ##### Efficiency
	Skip List provides functionality similar to an AVL Tree or a Red-Black Tree, which allows O(log N) search complexity as well as O(log N) insertion/deletion complexity within an ordered sequence of n elements. 


- ##### Concurrency
	Even though the time complexity is similar between Skip List and Balanced Tree, however, for a balanced tree, if we need to insert or delete a node, we might have to do a series of complex operation to rebalance the tree, and meanwhile we probably won’t be able to access other data. But in a skip list, if we have to insert a new node, things would be much easier and only the adjacent nodes will be affected, so we can still access large part of our data while this is happening. This advantage is quite useful when we need to be able to concurrently access our data structure. 

Therefore, Skip List fits perfectly for the requirement of this problem.

#### How Skip List works?
Basically, Skip List is a multi-level linked list. Take the below picture as an example, the lowest level is level 1, which connect all the nodes (5, 11, 20, 30, 68, 99) in the Skip List. The insertion of any node will have to make sure the elements in the list still ordered.

The height (number of levels) of each node is randomized when its inserted, and the Skip List implementation will make sure that if a node linked in a higher level, it will be linked all lower levels. For example, if the node has a height of 3, then it will be linked in level1, level2, and level3. One notice is, the height of node is not completely random, the goal is to ensure the number of nodes in higher level should be less than lower level. Final effect is like below picture, the number of level-3 node is less than level-2, whereas level 1 has all the nodes.

![enter image description here](images/search.png?raw=true)

**1. Search**
According to the structure of Skip List, to search a node is quite straightforward. We start from the head in the highest-level list one by one, until we meet the target node, or we meet some node larger than target, then we go down to the next lower level and keep looking. The reason why begin at the highest level is, nodes in higher level are sparser, thus we can move forward faster.

In the above example, when we need to find the node 68, we start from the head in level-3, the first node we met is 20 < 68, then we move to node 20 and keep looking forward. We find the next node 99 > 68, no need to move forward, then we go down to level-2. The node after node 20 in level-2 is 30, and 30 < 68, we move to node 30. Then we find 99 > 68 and go down to level-1, and find 68==68, there we are.

**2. Insert**
The Insert process is similar with search, with a bit more jobs. In the below case, we want to insert a number 80, the white node is our target node to insert. As the search operation, we start from the head in highest level. Only difference is that every time we go down to the next level, we record current node (As the red circled nodes).

In this example, we randomly give a height of 2 to the new node. Then we will need to update all the recorded nodes in the same or lower level with the new node. That is, node 30 in leve-2 and node 68 in level-1 will be pointed to node 80 respectively, and node 80 will be pointed to their old next node 99.
![enter image description here](images/insert.jpg?raw=true)

**3. Delete**

Delete a node is similar as well, we find the target node and record all the precursor nodes, then re-point the precursor nodes to the next node of target nodes. Below picture is an example.
![enter image description here](images/delete.jpg?raw=true)

**4. Rank**
The rank process happens during the search process, with an addition accumulation of the span between nodes. In Skip List, we can maintain a span value for each node in each level. The span equals the distance between the current node with its next. For instance, in level-3, we can go to node 20 from the header within one move but actually cross node 5 and node 11 as well, thus the span of header node in level-3 is 3. In the following case, when searching for 68, we can accumulate the span of all the node we met, which is 3+1+1=5, that is to say the node 68 ranked in the 5th place.

Similarly, we could also use this way to search node by a target rank, the trick is we compare the accumulated span with target rank to decide whether move forward or go down to next level when searching a node instead of comparing node’s value.

![enter image description here](images/rank.jpg?raw=true)

But the problem is, how to maintain the span value for every node? It’s easy to understand that the span of precursor nodes will need to be changed when insert or delete a target node, thus we can update them during the insert/delete operation.

Let’s look back the following picture of Insert operation. For the 3 precursor nodes (20, 30, 68), it’s not difficult to calculate their rank during the moving forward process. Then we can get the rank for the target node 80 `rank(80) = rank(68) + 1 = 6`. Then the new span of node 68 is easy to get, which equals `span_new(68) = rank(80) - rank(68) = 1`.

![enter image description here](images/insert.jpg?raw=true)

Then what about node 30? Same. `span_new(30) = rank(80) - rank(30) = 6-4 = 2`. One exception is the node 20, since we inserted a node between node 20 and node 99 but didn’t break their previous link, we need to add its span by 1.

So far, all the precursor nodes' span have been updated. Then we need to update the span value for the target node 80. It’s also quite straightforward, the trick is, we can use the old span value of precursor node to minus its new span value and plus 1. For instance, the span of node 80 in level-2 should be equals to `span(80) = span_old(30)-span_new(30)+1 = 2-2+1 = 1`.

The update for span value in delete operation is pretty similar, if we need to remove node 80, then the `span(20) = span(20) -1`, `span(30) = span(30) + span(80) - 1`, `span(68) = span(68) + span(80) - 1`.

#### Space Complexity
Since our design is an in-memory data structure, can we fit the whole Skip List into a server?
Let's make an assumption about the node size in the skip list:
> - pointer * 2 = 4bytes * 2 = 8bytes
> - span = 4 bytes
> - userId = 20 bytes
> - score = 4 bytes
> - rank = 4 bytes

Thus a node has roughly 40 bytes. Assume that we have average 10 levels for every nodes, then we need `10million * 10 * 40bytes = 4GB` memory in total, which shouldn't be a problem for a modern server.
	
### Leaderboards Service
Now, the description of how to implement a Skip List is done. We can consider how to utilize it for our leaderboard services.

Suppose that our SkipList Implementation have the following methods
- void insert(Node){}
- void delete(Node){}
- int getRankByNode(Node){}
- Node getNodeByRank(int){}

Then we can address these methods in the following code
```java
class LeaderBoard{
	//map from userId to LeaderBoardRow
	private Map<String, LeaderBoardRow> cache;
	//Skip List of LeaderBoardRow
	private SkipList<LeaderBoardRow> skipList;

	public LeaderBoard(){
		cache = new HashMap<>();
		skipList = new SkipList<>((a, b)-> {
			if (a.getScore() == b.getScore()){
				//if users have same score, than rank them by their userId lexicographically.
				return a.getUserId().compareTo(b.getUserId());
			} else {
				// we can change the score ordering here (increasing or decresing)
				return a.getScore() - b.getScore();
			}
		});
	}
	public void updateByUserId(String userId, int newScore){
		if (cache.containsKey(userId)){
			LeaderBoardRow lbr = cache.get(userId);
			lbr.setScore(newScore);
			skipList.delete(lbr);
			skipList.insert(lbr);
		} else {
			LeaderBoardRow lbr = new LeaderBoardRow(userId, newScore);
			cache.put(userId, lbr);
			skipList.insert(lbr);
		}
		// note that we don't update rank in this step
		// since in SkipList, insert or delete one row may cause many rows' rank affected.
		// instead we calculate rank for a certain row when fetch it.
	}

	public LeaderboardRow fetchByUserId(String userId) throws UserNotExistException{
		if (cache.containsKey(userId)){
			LeaderBoardRow lbr = cache.get(userId);
			int rank = skipList.getRankByNode(lbr);		// recalculate rank
			lbr.setRank(rank);
			return lbr;
		} else {
			throw new UserNotExistException();
		}
	}

	public LeaderboardRow fetchByRank(int targetRank) throws RankNotExistException{
		if (targetRank>skipList.size() || targetRank<0){
			throw new RankNotExistException();
		} else {
			LeaderBoardRow lbr = skipList.getNodeByRank(targetRank);
			lbr.setRank(targetRank);	// reset rank
			return lbr;
		}
	}
}

class LeaderBoardRow{
	private String userId;
	private int score;
	private int rank;

	public LeaderBoardRow(String userId, int score){
		this.userId = userId;
		this.score = score;
		this.rank = -1;
	}

	//.. getter and setter
}
```

## Other Solutions
### 1. RDBMS
We can store the userId and score into a user_score table, then we can implement the 3 API through simple SQLs. 
For example, we implement the fetchByUserId() by the following sql.
```sql
SELECT a.user_id, a.score, a.rank
FROM (SELECT user_id,
             score,
             RANK() over (ORDER BY score) as rank
        FROM user_score
      ) a
WHERE a.user_id = userID
```

- Pros
	- Simple. We can utilize the power of SQL, no need other complex logic.
- Cons
	- When the data amount is huge, the efficiency would be a serious problem.

### 2. Self-Balanced Order-Statistic Tree (SBOST)
As mentioned above, we can also use a self-balanced order-statistic tree such as AVL Tree or Red-Black Tree to design the leaderboard. SBOST is a particular kind of a binary search tree. A binary search tree stores node and each node must be greater than all nodes in its left subtree, and not greater than any in the right subtree. For being balanced, a binary search tree must fulfill another condition: the difference between the heights of the left and right subtrees of any node must be at most 1. The height of a subtree is the maximum number of jumps between the root of the subtree and its deepest leaf. Being balanced has a critical repercussion: considering the binary layout of the tree, as well as the relation of each node with its two subtrees, all the basic operations insertion, deletion, and search can be achieved in O(log N) time complexity.

In addition, being an order-statistic tree means that it supports two additional operations: selection and ranking. The first one refers to finding the element ranked in a given rank place, whereas the second one refers to finding the rank of a given element in the tree. Both can also be performed in O(log N) when a self-balancing tree is used. However, for doing so, all nodes must store one additional attribute, which is the size of the subtree starting at that node. In other words, it refers to the number of nodes below and including it. All operations that modify the tree must consider this attribute and preserve the relation presented in the following without altering the time complexity: 
`node.size = node.left.size + node.right.size + 1`

- Pros
	- High Efficiency, overall time complexity is similar to Skip List.
- Cons
	- The tree structure is much harder to implemented than Skip List.
	- The concurrency problems come in when the tree is modified and needs to rebalance. The rebalance operation can affect large portions of the tree, which would require a mutex lock on many of the tree nodes. Inserting a node into a skip list is far more localized, only nodes directly linked to the affected node need to be locked.

### 3. Redis Sorted Set (ZSET)
Redis Sorted Sets are, similarly to Redis Sets, non repeating collections of Strings. The difference is that every member of a Sorted Set is associated with score, that is used in order to take the sorted set ordered, from the smallest to the greatest score. With sorted sets we can add, remove, or update elements in a very fast way. Since elements are taken in order and not ordered afterwards, we can also get ranges by score or by rank (position) in a very fast way. The overall time complexity is O(log N).

The characteristic of Redis Sorted Set makes it perfect for building a leaderboard service, where every time a new score is submitted we can update it using ZADD. We can easily take the top users using ZRANGE, and given a userId to return its rank using ZRANK.

- Pros
	- Can be utilized directly without extra implementation.
	- Based on Redis, have all its advantages such as High efficiency, scalability, reliability and great fail over capabilities.
- Cons
	- The only drawback of ZSET is, it can only sort the element by score. Thus it is not so extendable if we want to change the ranking condition to a more complex one.

### 4. Buckets
If there is a score boundary, then we could use Bucket Sort to get the rank. We can make a bucket for each score and store the frequency of the score in this bucket. Then we can get the rank by calculate the frequency of all higher or lower scores. But this way has a limitation, the score range cannot be quite large, otherwise the efficiency would be fairly bad and consume too much memory.

However, if we don’t have a restrict expectation of the precision of rank, then another solution is that we can change the bucket a little bit to save the user score into buckets based on different score ranges. In this way, we can reduce the bucket number and improve the efficiency a lot, which works even if the total score range is huge. For instance, assume that the scores are distributed within the range [0, 99], we can set 4 buckets, each buckets need to store the frequency and the score range. Take the following diagram as example.

![enter image description here](images/bucket.jpg?raw=true)
In this way, we can predict the rank of a given score. For instance, if a user have score 60, we will be looking at the bucket [50, 74], and use the frequency of this bucket (42) and the highest rank of this section ( score 74 ranked at 5th place) to calculate the target rank by this formula: 
`Rank = 5 + 42 * (74-60)/(74-50) = 30`

For the highest rank of each bucket, we can simply get them by calculating the frequency of all higher or lower buckets. The time complexity of findRank in this algorithm is O( C ) where C is the number of buckets, and setScore is O(1). Or, we can also set a scheduled task to constantly scan all the buckets and pre-generate their highest rank, which will reduce the findRank to O(1) time complexity.

Pros: Extremely efficient.
Cons: The rank we got could be imprecise, especially when the scores are not uniformly distributed.


## Conclusion
There is no such thing as a best design, Skip List also has its limitation. For instance, the Skip List takes more space than similar approach such as Self-Balanced Order-Statistic Tree because it has multiple layers for most nodes, which consumes a certain large amount of extra memories. This shortcoming could be more serious when the user number grows to even larger.

But based on the requirement of the given problem, I think Skip List should excel in terms of efficiency, concurrency and simplicity among all the possible solutions.
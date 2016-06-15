# Degree of Separation on Twitter
    
I did this as part of the hiring challenge for a company. I think it's
cool and people can make it better. Pull requests are welcome. :)

Using the scripts requires `tweepy` and `numpy`. Please use the
`requirements.txt`:
    
```shell
        pip install -r requirements.txt 
```
    
A few things that aren't right about the script:
    1. Exceptions are caught away from where they occur.
    2. Because of the way twitter returns users, we now care only
       about the 5000 followers/friends returned in one request. This is
       good sometimes because some users have millions of followers fetching
       all of which would exhaust our API limit because we can only fetch 5k
       at a time.
    3. It is like a hacked up thing. 
       
    
## Problem 

We need to find the degree of separation between two users on
    Twitter, i.e.  the number of intermediate twitter users
    encountered as we follow the friend links from the first user and
    move towards the second user.  The twitter REST API refers to the
    people a user follows as his/her friends and the ones that follow
    him/her as the followers. So, for a user we have outbound friend
    edges and inbound follower edges in the twitter social graph.
    

 **Note**: The problem statement says that we must find the degree of
 separation between two users. There have been similar
 experiments done in the past where degrees of separation were
 calculated for large graphs like Facebook[1]. But it is to be
 noted that in those experiments, the degree of separation for
 a person is the *average* number of nodes that separate him
 from *any* other person in the graph. Since, this calculation
 requires calculating the average number of intermediate nodes,
 the problem is then translated into the problem of finding the
 unique neighbors of a node in the first level, the second
 level and so on. To approximate the number of unique friends
 of a user in a specific level, Flajolet-Martin's algorithm[2]
 has been used. Once we have these numbers, calculating the
 average degree of separation for a user is simply computing
 the average of these unique node numbers weighted by the level
 number. For example, if Alice has 3 friends and 6
 friends-of-friends, then the average degree of separation for
 Alice will be = (/ (+ (* 3 1) (* 2 6)) 9.0)
 [= 1.6666666666666667 ].

But the problem we are trying to solve is different.
The problem we are trying to solve has been solved in a
research paper[3]. The findings of the paper were that a
bidirectional search is the most effective strategy for
accomplishing this task. 

I have written a script implementing the accurate solution.
It is in the same directory as this file: `digsep.py` 

And a script implementing the probabilistic search mentioned
in the paper[3]: `digsep_probabilistic.py`.

Because of the rate limiting that twitter imposes, I think we
can only run this script once or twice in 15 minutes. We are
allowed 15 API calls in a 15 minute window.

Here is a session:

```shell
        --> Opening in your browser: https://api.twitter.com/oauth/authorize?oauth_token=pPukIAAAAAAAvUfKAAABVOkLqrk
        --> Please copy the PIN here after you authorize.

        Verifier>  2388715
        1295560122-PMd82QNErD16ku0wyxr38eze7hR8fLZqQEg9H8f
        Source User's handle> paulg
        Destination User's handle> narendraj9
        Begin: Paul Graham
        End: Narendra Joshi
        ................Found!
        Paul Graham --> Jennifer 8. Lee --> Erlich Bachman --> Siddha Ganju --> Narendra Joshi

```
    
**Addendum** : I hadn't thought about it before but we can implement
    a distributed crawler[4] for twitter and build the social
    graph. Once we have the social graph, the same algorithms can be
    applied to it and then we won't incur the cost of networking. So,
    this can be thought of as the pre-processing step. 
        
## Design and Implementation

We need to take into consideration the following facts:

1. Making an API call is many orders of magnitude more expensive
compared to a CPU cycle. So, it is very important that we try
keep the number of API calls to a minimum.

Moreover, Twitter rate-limits requests to about 15 per
rate-limit window.  The rate limit window is 15 minutes.

2. The branching factor for the twitter social graph is huge. So,
we should try to reduce the depth of a *single* traversal into
the graph. This justifies going for a bi-directional search.

For example, if the twitter social graph is just a binary tree
then for a degree of separation of 6, the bi-directional search 
is around 4 times better than the uni-directional search as
`2 * 2^3 < 2^6`. For an n-ary tree, bi-directional is ~ n^3 times
better. [#TODO Try figuring out the branching factor for twitter]


This is a high-level overview of the algorithm we follow:

For an optimal solution, we carry out two breadth first searches--
one from the source and the other from the destination. We try to
find an intersection between the regions covered by the two
searches. Once a node is found at the intersection, we can build
the path from the source node to the destination following parent
edges from the node at the intersection.

For a non-optimal but faster solution, we can ditch bread-first
search and try to increase the number of nodes we add to our
explored territory in each step. We could try to pick the node on
the perimeter with the highest out-degree. This increases our
chances of reaching the destination node but not through the
optimal path. We expect this because the twitter social graph is a
densely connected graph. 
   
### Algorithm (Optimal)

See the script `digsep.py` for an implementation of the above 
mentioned algorithm for an optimal solution. 

Explanation:
The Frontier class keeps track of the territory of nodes that we have
explored with a Breadth-First Search from a node. We keep information
about the nodes that lie on the perimeter of this territory. We pick
one of nodes closest to the source and expand the territory following
its outbound edges and update the perimeter with newly found nodes.
On every expansion of our realm, we check if the perimeters of our source
node and destination have any element in common. We do the same thing for
the destination node. I think the source code clearly explains the details.
I have tried to keep it readable.

### Algorithm (Probabilistic)
    
Bi-directional breadth first search is optimal but a greedy
approach that aims at growing the explored region of the graph can
produce results faster and probably with a lower number of API
calls.

The idea is that when we are strictly doing Breadth-First search,
we are constantly making sure that the nodes on the periphery
always remain at almost the same distance from the source,
i.e. their distances from the source node always differ by not
more than unity. This can be slow when the degree of separation is
large because we have to always reach there by growing a circle of
explored nodes around the source. If you think about it, making a
circle is such a waste of effort because the destination node is
only in one direction and we move equal amounts in all of the
directions possible.

We can relax ourselves a bit and try to be a little greedy but
picking up a node for expansion on the periphery that has the
maximum number of outgoing edges. But we would like to bias
ourselves to following BFS, so we pick nodes with a probability
distribution that favors the nodes closer to source, i.e. the
probability of our picking a node decays exponentially as we move
further away from the source. 

This approach has been implemented in `digsep_probabilistic.py`.
This is a straight forward implementation of the algorithm mentioned
in the paper.

## References

1. [Facebook: Three and a half degree of separation](https://research.facebook.com/blog/three-and-a-half-degrees-of-separation/)
2. [Wikipedia: Flajolet-Martin Algorithm](https://en.wikipedia.org/wiki/Flajolet%E2%80%93Martin_algorithm)
3. [Degrees of Separation in Social Networks](https://www.aaai.org/ocs/index.php/SOCS/SOCS11/paper/viewFile/4031/4352)   
4. [Crawling Billions of Pages](http://engineering.bloomreach.com/crawling-billions-of-pages-building-large-scale-crawling-cluster-part-1/)       
       

    
    

    
    

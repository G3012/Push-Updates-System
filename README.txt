Data Structures:-

- struct action
    - user_id 
        - id of the user who performed the action
        
    - action_id 
        - id of the action (either a post or a like or a comment)
        - each action of a particular user has a particular action_id associa
        
    - action_type  
        - 0 for post, 1 for like, 2 for comment
        
    - priority_val 
        - priority value of the action
        
    - timestamp
        - time when the action is performed


- struct feed_compare
    - this is a functor (a class with an overloaded 'operator()') used later to sort queues according to their priorities


- class node
    - int user_id
        - id of the user (initialization step has input id given)
        
    - bool order
        - 0 represents priority based on # of common friends, 1 represents priority based on chronological order
        - order = rand() % 2 (initialization step gives order randomly)
        
    - queue <action> wallQueue
        - wall queue of each node is implemented as a normal queue data structure.
        - Queue containing actions of this user
        
    - priority_queue <action, vector<action>, feed_compare> feedQueue
        - feed queue of each node is implemented as a priority queue (internally as a vector) with comparision operator defined as feed_compare (common friends based or chronological based)    
        - Queue containing neighbour's actions (which come up on the feed of this user)

    - vector<int> numActions
        - 3 vectors of integers having the count of number of actions of each type
            - numActions[0] - post count
            - numActions[1] - like count
            - numActions[2] - comment count


- map <int, vector<int>> graph
    - we store the entire graph in this data structure
    - it is a map of type <node_id, all_nodes_neighbouring_to_node_id (adjacency_list)> 
    * using this type of data structure for graph makes it easy for a user_id to obtain list of neighbours and no.of neighbours


- vector <node> users(NUM_NODES, node(0));
    - this creates a vector of size NUM_NODES, each initialized with the values of node(0) (node(node_id) is a constructor declared in class node)
    - note that this does not contain the list of neighbouring nodes of the user_id, to obtain the list of neighbouring nodes, we need to use the above graph


- queue <action> actionQueue
    - queue of actions (shared between userSimulator and pushUpdate)
    - large queue containing the list of all actions generated by all users (by the userSimulator)


- queue <int> userQueue
    - queue of users whose feed queue gets updated (shared between pushUpdate and readPost)
    
    
    
Queue size rationalization:-

* considering average case calculations based on given data
* average no.of friends for a user = 289,003/37,700 ≈ 8

- wallQueue
    - wallQueue size never decreases
    - a user selected among the 100 who generate actions may increase his wallQueue size by 12(avg no.of actions)
    - if the user is not selected the wallQueue remains the same
    

- feedQueue
    - each user has 8 friends 
    - considering all of them generate actions (12 each)
    - so, the avg max size of feedQueue is 12*8 = 96 


- actionQueue
    - The number of actions generated by each user = 3 * log2(2 * no.of friends)
    - i.e total no.of actions generated = 100 * 3 * log2(2 * 8) = 1200 actions
    - therefore, the avg size of actionQueue is 1200


- userQueue
    - 100 users generate 12 actions each and have 8 friends
    - considering among these 100 users, no two users have common friends
    - then the avg max size of userQueue = 100 * 12 * 8 = 9600
    
    
Lock usage:-

- we are using 1(actionQueue) + 1(userQueue) + 37,700(one for each feedQueue in every user) = 37,702 locks (n + 2 locks)

- actionQueue lock
    - it is used for prevention of race condition between userSimulator thread and pushUpdate threads
    - it also prevents race condition among the pushUpdate threads 

- feedQueue locks 
    - There are n feedQueue locks, each dedicated to a node's feedQueue.
    - each user has a lock for their feedQueue to prevent race conditions between pushUpdate threads and readPost threads.
    - pushUpdate/readPost threads use them whenever they need to access/update a user's feedQueue.

- userQueue lock
    - this is locked to prevent race conditions between pushUpdate threads and readPost threads.
 

* Parallelism / Concurrency: the locks used enable parellelism as multiple threads can work simultaneously without having race conditions with each other. This can be seen in the manner the sns.log file is printed.
    - The userSimulator thread locks the actionQueue when it needs to push new actions; this queue is also used on a shared basis with the 25 pushUpdate threads, which would not be able to access it till the userSimulator gives up the lock and vice versa. userSimulator locks the actionQueue lock once per action of a chosen node and unlocks it after signaling available pushUpdate threads. But, once given access to the queue, the userSimulator can lock the queue again once it is scheduled by the scheduler, while the pushUpdate threads push the actions to neighbour feedQueues (parellel pushing). Therefore, both the userSimulator thread and pushUpdate threads can be active simultanously.
    - Similarly, the 25 pushUpdate threads can parallely work on different feedQueues simultanously as each feedQueue has its own lock. The same applies for the 10 readPost threads. Moreover, just like the shared relationship of actionQueue between pushUpdate and userSimulator threads, there is a shared relationship of userQueue between pushUpdate and readPost threads. Therefore, many readPost threads can, after popping a user from the userQueue, output feeds simultaneously.

# CBS-2021

## Running the code

If you'd like to use this code, please:
1) compile it (for example, ```g++ main.cpp map.cpp agent.cpp ctNode.cpp ctSolution.cpp search.cpp -o try-multirobot -std=c++0x```),
2) specify the parameters in [call-multirobot.py](call-multirobot.py),
3) and run it with python3.

The code uses maps and scenarios from [Moving AI Lab](https://movingai.com/benchmarks/mapf/index.html).

An example of a map can be found in the file [check.map](check.map) (same with examples of scenario and goal files: [check.scen](check.scen), [check.goals](check.goals)). 
In call-multirobot.py, one can choose which optimizations are going to be tested and enter the parameter values. 
The variables include:
- `path_to_bin` (path to the executable), 
- `path_to_map` (path to the map),
- `paths_to_scen` (paths to scenarios), 
- `path_to_goals` (path to the file with additional goals),
- `MAX_AGENTS` (the maximum number of agents),
- `MAX_SCEN` (the maximum number of scenarios), 
- `MAX_FAILED` (maximum consecutive failures, after which this scenario will not be included in tests with more agents), 
- `TIME_OUT` (the time limit in seconds), 
- `dijkstra_precalc` (set true to use exact heuristic precomputation),
- `use_CAT` (set true to use the conflict avoidance table), 
- `heuristic` (choose heuristic: normal, normal_diagonal, number_of_conflicts, number_of_conflicting_agents, number_of_pairs, vertex_cover), 
- `prioritize_conflicts` (set true to prioritize cardinal and semi-cardinal conflicts),
- `use_bypass` (set true to use the bypass technique),
- `use_ecbs` (set true to use suboptimal CBS),
- `omega` (choose suboptimality factor for ECBS),
- `use_symm` (set true to use symmetry breaking constraints),
- `online` (choose an online setting with many goals or an offline setting with one goal),
- `horizon` (conflicts will be ignored after this number of steps),
- `replanning` (the number of timesteps before replanning),
- `print_paths` (set true to print paths for each agent).

## More on file formats

All maps begin with the lines:

```
type octile
height y
width x
map
```

where y and x are the respective height and width of the map.

The map data is store as an ASCII grid. The upper-left corner of the map is (0,0). The following characters are possible:

- `. G S W` - passable terrain,
- `@ O T` - out of bounds or unpassable terrain.

A scenario is a text file with a list of start and finish points for each agent. Each line of a scenario has 9 fields: bucket, map, map width, map height, start x-coordinate, start y-coordinate, goal x-coordinate, goal y-coordinate, optimal length. The optimal path length is assuming sqrt(2) diagonal costs.

A goals file contains additional goals for agents in the online setting. Every line contains a list of goal x-coordinates and goal y-coordinates, followed by -1.

## Code structure

This repository contains the following files:

#### [agent.cpp](agent.cpp)

A file describing the Agent class. `Agent::Agent(int agentId)` describes how an agent is initialized: its starting point (start_i,start_j) equals (-1,-1), the function receives its id as a parameter and its speed equals 1. `Agent::getAgent(std::ifstream& agentFile, bool online)` is a helper function for reading agent parameters from a scenario file. If `online == true`, the goal file is parsed. Otherwise, only one goal point is read from the scenario file. The agent's speed is chosen randomly.

#### [agent.h](agent.h)

A file describing the agent class. The type of fin_i and fin_j is `std::vector<int>`: while each agent has only one starting location, it can have multiple goal locations.

#### [call-multirobot.py](call-multirobot.py)

A file to make running the executable with different parameters easier. Sets up a timer for each map-scenario combination, kills the MAPF solver, if it takes too much time. For each number of agents from 1 to 100, the algorithm is tested on 25 scenarios. Then, the percentage of successfully executed scenarios (success rate) is counted.

#### [check.goals](check.goals)

A dummy goals file for two agents.

#### [check.map](check.map)

A dummy 4x5 map.

#### [check.scen](check.scen)

A dummy scenario file for two agents.

#### [constr.h](constr.h)

A file describing Constraint and Conflict structures. A constraint has the following fields:
- `std::string type`,
- `int agent`,
- `int time`,
- `std::pair<int, int> v1`,
- `std::pair<int, int> v2`.

There are two types of constraints: a vertex constraint, where the agent is not allowed to be in v1 at a certain time, and an edge constraint, where the agent is not allowed to be in vertex v1 and try to reach vertex v2 at a certain time. Barrier constraints consist of vertex constraints.

A conflict has the following fields:
- `std::string type`,
- `std::pair<int, int> agents`,
- `int time1`,
- `int time2`,
- `std::pair<int, int> v1`,
- `std::pair<int, int> v2`.

There are four types of conflicts: a vertex conflict, where agent[0] and agent[1] collided in v1 at a certain time; an edge conflict, where agent[0] was traveling from v1 to v2 at a certain time, agent2 was traveling from v2 to v1; type none conflict (no conflict) and a rectangle conflict.

#### [ctNode.h](ctNode.h)

A file describing the CTNode class, a class for constraint tree nodes. There are some helper types and structures: `VertexConstrStruct` for vertex constraints; `EdgeConstrStruct` for edge constraints; `Path` for paths (each step is a `std::pair<int, int>`); `StateMap`, an unordered map, where the key is a tuple (an agent's state, `i j t`) and the value is a set of agents in that state; `KeyFour`, `KeyFourHash`, `KeyFourEqual` and `pair_hash` for hashing and comparison purposes; `EventVertex` and `EventEdge` for fast conflict detection.

#### [ctNode.cpp](ctNode.cpp)

A file describing the CTNode class. It has the following fields:

- `std::vector<Path> paths`, a vector of paths for each agent,
- `int cost`, the cost of this solution,
- `int maxTime`, a constant used to create constraint structures,
- `VertexConstrStruct vertexConstr`, vertex constrains, imposed on the agents in this CTNode,
- `EdgeConstrStruct edgeConstr`, edge constrains, imposed on the agents in this CTNode,
- `ConfMap conflictAvoidanceTable`, to count how many agents were in each state,
- `StateMap stateAgentMap`, to store which agents were in each state,
- `bool** graph`, for the vertex cover heuristic.

These are the CTNode functions:

- `bool isCover(int V, int k, int E)`, for the vertex cover heuristic,
- `int findMinCover(int n, int m)`, for the vertex cover heuristic,
- `void insertEdge(int u, int v)`, for the vertex cover heuristic,
- `void countCost(std::string heuristic)`, for counting the solution cost,
- `void countCAT()`, for filling in the CAT,
- `int countPathCost(int i, std::string heuristic)`, for counting the cost of a single path,
- `std::string findConflictType(Map& map, bool dijkstra, std::map<pairVert, int, pvCompare>& distMap, std::vector<Agent>& agents, Conflict& conflict, std::string heuristic)`, a function receiving a conflict and returning its type: cardinal, semi-cardinal or none,
- `Conflict findBestConflict(Map& map, bool dijkstra, std::map<pairVert, int, pvCompare>& distMap, std::vector<Agent>& agents, bool prioritizeConflicts, bool useSymmetry, std::string heuristic, int horizon)`, a function for finding the best conflict to split on,
- `int findNumOfConflicts(Map& map, std::vector<Agent>& agents, int agentCheck, int horizon)`, a function counting the number of conflicts for some optimizations and heuristics,
- `bool operator< (CTNode &other)`,
- `bool operator== (CTNode &other)`,
- `bool operator!= (CTNode &other)`.

#### [ctSolution.cpp](ctSolution.cpp)

A file describing the CTSolution class. It has the following fields:

- `std::priority_queue<CTNode, std::vector<CTNode>, CompareCBS> heap`, a priority queue for choosing the best CTNode,
- `Map map`,
- `std::vector<Agent> agents`,
- `std::map<pairVert, int, pvCompare> distMap`, a structure used to store distances between each pair of empty cells in exact heuristic precomputation,
- `bool useDijkstraPrecalc`, set true to enable exact heuristic precomputation (this algorithm is very time-intensive and should be used on smaller maps),
- `bool useCAT`, set true to enable CAT,
- `std::string heuristic`, choose heuristic: normal, normal_diagonal, number_of_conflicts, number_of_conflicting_agents, number_of_pairs, vertex_cover, 
- `bool prioritizeConflicts`, set true to prioritize cardinal and semi-cardinal conflicts. Splits will be done on cardinal, non-cardinal and other conflicts, in this order,
- `bool useBypass`, set true to look for helpful bypasses and reduce the number of nodes in the constraint tree,
- `bool useFocal`, set true to use focal search (for suboptimal CBS),
- `double omega`, if focal search is enabled, the solver will return solutions with cost <= optimal cost * omega,
- `bool useSymmetry`, set true to use symmetry breaking constraints. Only works with Manhattan distances,
- `bool online`, set true to use replanning and read goal locations from the goal file,
- `std::vector<std::vector<std::pair<int, int>>> goalLocs`, the goal locations for the online setting,
- `int horizon`, conflicts will be ignored after this number of steps (works in both offline and online settings),
- `int replanning`, paths will be replanned after this number of steps (for the online setting),
- `bool printPaths`, set true to printed out paths for each agent.

The main function is `solve()`. It runs the precomputation and is in charge of running MAPF instances - one instance in the offline setting and many instances in the online setting. It prints the cost of the solution and valid paths, if `printPaths` is set to true. The other functions are

- `std::map<pairVert, int, pvCompare> dijkstraPrecalc(Map& map)`,
- `Path lowLevelSearch(CTNode node, int i)`, the low level of CBS which returns paths for a single agent,
- `std::vector<Path> highLevelSearch()`, the high level of CBS which chooses nodes in the constraint tree and imposes constraints on the agents.

#### [ctSolution.h](ctSolution.h)

A file describing the CTSolution class. Includes a comparator for comparing constraint tree nodes by cost.

#### [main.cpp](main.cpp)

The main file of this code. It parses the parameters and the files. There are two functions:
- `std::vector<Agent> readAgents(std::ifstream& agentFile, int size, bool online)` and
- `std::vector<std::vector<std::pair<int, int>>> readGoals(std::ifstream& goalFile, int size)`
for parsing the agents' starting and finishing locations from the scenario and the goals files.

#### [map.cpp](map.cpp)

A file describing the Map class. It has the following fields:


int height;
int width;
int** grid;
std::string metricType;
int hweight;

Map();

void getMapOptions(std::ifstream& mapFile);
void getMapGrid(std::ifstream& mapFile);
bool cellOnGrid(int i, int j) const;
bool cellIsTraversable(int i, int j) const;

#### [map.h](map.h)

#### [pairVert.h](pairVert.h)

#### [search.cpp](search.cpp)

#### [search.h](search.h)

#### [searchNode.h](searchNode.h)


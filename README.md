# QuickMatch
A search algorithm to efficiently find matches between huge sets of (real estate) properties and search criteria.

### What's the requirement?
An efficient algorithm that helps s determine a list of matches with match percentages for each match between a huge set of properties (sale and rental) and buyer/renter search criteria as and when a new property or a new search criteria is added to the system by an agent.  

#### Detail
The system has a lot of properties from property sellers and search requirements from property buyers which get added every day. These multiple properties and search criteria get added through an application by agents. We need an algorithm to match these properties and search criteria as they come in based on 4 parameters such that each match has a  match percentage.

The 4 parameters are:
- Distance - radius (high weightage)
- Budget (high weightage)
- Number of bedrooms (low weightage)
- Number of bathrooms (Low weightage)


- Each match should have a percentage that indicates the quality of the match. Ex: if a property exactly matches a buyers search requirement for all 4 constraints mentioned above, it’s a 100% match.  
- Each property has these 6 attributes - Id, Latitude, Longitude, Price, Number of bedrooms, Number of bathrooms
- Each search criterion has these 9 attributes - Id, Latitude, Longitude, Min Budget, Max budget, Min Bedrooms required, Max bedroom reqd, Min bathroom reqd, Max bathroom reqd.

###### Functional requirements
- All matches above 40% can only be considered useful.
- The code should scale up to a million properties and requirements in the system.
- All corner cases should be considered and assumptions should be mentioned
- Requirements can be without a min or a max for the budget, bedroom and a bathroom but either min or max would be surely present.
- For a property and requirement to be considered a valid match, distance should be within 10 miles, the budget is +/- 25%, bedroom and bathroom should be +/- 2.
- If the distance is within 2 miles, distance contribution for the match percentage is fully 30%
- If the budget is within min and max budget, budget contribution for the match percentage is full 30%. If min or max is not given, +/- 10% budget is a full 30% match.
- If bedroom and bathroom fall between min and max, each will contribute full 20%. If min or max is not given, match percentage varies according to the value.
- The algorithm should be reasonably fast and should be quick in responding with matches for the users once they upload their property or requirement.

### Problem analysis

> __"This is a search problem"__

A criterion or a property can have matches against any record in the system and the number of records can be humungous.
The algorithm should be a finite (terminating) function which deffinately terminates with a set of rich matches. The set of matches can be an empty set indicating **no match found**.

To accomplish this we need to:
- search
- match

While matching can be implemented efficiently and has scope only for micro-optimisation **search** is the subtask which has major scope of effective optimisation.

### The approach
One of the best ways to do faster search is to reduce the search space.
Naively this can be done in O(N) where N is the total number of records in the system i.e. properties while searching with criterion and criteria while searching with property.

But to do it faster the search should be narrowed down to a subset of the universal set of records hereafter referred to as **U**.

The entities of **U** are complex objects with various attributes (which might also evolve). Search space reduction in this type of data needs to happen at multiple levels so as to have minimum query time. The aim is to have as less records as possible for match calculation. 

Analysing the attributes of both the types Property and Search Criterion there are a few indexable candidates which can be beneficial in eliminating records which the query is not interested in. Few of the effective attributes which might contributes to faster search are:
- Price/Cost
- latitude and longitude
- Number of bed/bath rooms

Each of these bear a weightage out of which Price and geolocation have high weightage. These attributes also come with toleraces which might be useful in cases when there is not enough search results to display or in advanced search.

The records in **U** need to be indexed on, atleast, the price and geolocation attributes which aid faster queries and insertions. But the order in which these indexes are evaluated play a great role in the execution time of a query.

The most effective order of applying indexing according to the type of the attributes or the records is in the following order:
- Index by Geo location (latitude + longitude combined)
- Index by price
- Index by number of bedrooms/bathrooms

### Matching process steps
The action of adding a new search criterion or a new property to the system will trigger the process of matching which is basically a _search for matches._ Therefore referring to the process of finding matches as "search" hereafter.
Everytime a search is performed the system will do the following in sequence:  
**Step 1**: Eliminate every record which is not close (within 10 miles) to the supplied lat-longs.  
**Step 2**: Eliminate every record which is not within (+- 25%) the supplied price range.  
**Step 3**: Eliminate every record which does not has no. beds/baths close (+-2ish) to the supplied number.  
**Step 4**: Perform match calculation on the remaining records and return the results.  

The above steps are applied in succession i.e. every operation is applied on the output from the previous step

### Data arrangement (partitioning)
The data can, logically, be arranged in the following format:
- Every record belongs to a cell in the Geo grid.
- Within each grid cell, there a buckets of price ranges. (ex: $10,000 - $20,000) and every record belongs to one of these price buckets.
- Within each price bucket, there are buckets of number of beds and every record belongs to one of these bedCount buckets.

### What is Geo grid?
To perform efficient search (or search space reduction) according to the geospatial attributes, the data needs to be sortable or hashable by the geo data point.
To achieve this we can overlay an imaginary grid on the real world area of operation (say The US). Therefore every record in the system (with lat/long info) now starts belonging to a cell in this grid. Let's call this grid as Geo Grid.
This helps us achieve partitioning of data by their lat/long attributes. Trading off memory now we have a constant time lookup for an arbitrary lat/long as to which cell it belongs to in the grid. This information can help us quickly narrow down our search to a **_very few cells_** stripping out > ~99% of the search space in constant time. (considering a system with millions of records spread of millions of square miles)

> **How many are 'very few cells' to consider?**
>This depends on how the area(a Country, etc) is partitioned i.e. the size of each cell of the grid matters here. If carefully decided in accordance with the business domain search constraints (result > 10 miles away are invalid) this can result in better performance. Too big or too small (w.r.t. business search constraint) grid cell size can lead to improper narrowing down and might end up spitting out more (and farther) records than ideally required.

Consider this mock representation of a Geo Grid (obviously figure is not to scale) where India is the operational region and the land is divided into numerous small grids. Each grid contains data arranged to assist fast search and querying.
![Geo Grid on India](https://raw.githubusercontent.com/jskrnbindra/QuickMatch/master/assets/grid_india.jpeg "Geo Grid on India")    

##### Contents of each cell in the Geo Grid
Each grid cell has data sorted in price ranges where each price range bucket has data sorted according to (finite) number of bedrooms buckets where each bedCount bucket has raw Property objects which can then be filtered based on finer search criteria (ex: no. of bathrooms).  
![Data arrangement view](https://raw.githubusercontent.com/jskrnbindra/QuickMatch/master/assets/data_arrangement.jpeg "Data arrangement")

#### Why is it better to do SSR by Geo Location first?
Say a cell in the Geo Grid represents a land area of **10 square miles**.  
**10 mi<sup>2</sup> = 2.788 * 10<sup>7</sup> ft<sup>2</sup>**  
Let's say we have each plot of 2500 ft<sup>2</sup> on an avergae.  
**=> (2.788 * 10) / 2500 = ~11000**  
Let's say each plot has ~10 properties (flats/apartments).  
**=> 11000 * 10 = ~110000**  
Therefore an area of **10 mi<sup>2</sup>** can have at max **~110K properties**. We've not considered space for roads, parks and everything other than houses.   
This is the approximate upper limit of how many records we'll get based on the bucket we choose from the Geo grid. This ~110K properties are again bucketed inside these grid cells.

### Complexity analysis
**Step 1**: All it involves is looking up the lat/long in a Geo hash grid and finding out the neighbouring cells in the grid. This can be done in constant time. Therefore let's say this step narrows down the search to **G** buckets.  
**Step 2**: This involves extracting out records whose price fall within the specified range from a list of records sorted by price-ranges. Binary search can be done on **G** records to accomplish this. Let's say this step narrows down the search to **P** buckets.  
**Step 3**: This involves looking up in **P** buckets of records according to number of beds. This can be done in constant time for each bucket. Let's say this step narrows down the search to **B** records.  
**Step 4**: Perform matching against all the records. This is constant for each record. And we finally get an output of **M** records with match percentages.  

**Total time complexity: O(1) + O(log<sub>2</sub>G) + O( P ) + O(B)**

### Will it scale?
Price and Location bearing high weightage and significant contributors to Search space reduction are placed first in the list. 
Geolocation is the best candidate to perform Search Space Reduction first because no matter how many properties or search criteria get added to the system there is always an upper bound on how many properties can exist in a fixed area (size of each Geo grid cell). Therefore the output of the first step will grow linearly with density of that area but after the area getting saturated it becomes constant. 
Thus no matter how much the system scales, there is always an upper bound to the search space and thus an upperbound to the query time.

**Step 2** involves binary search which is also a SSRT(Search Space Reduction Technique) and is efficient enough to find the result in just **O(logN)** (i.e. just ~30 steps to find a value out of 1 billion records). Therefore, this will keep performing pretty fast even if we have searches for densly populated areas.

Following that is again constant time lookups to get rid of records which do not have expected number of bedrooms. In the worst case this will not exclude any records.

### Is it flexibile?
The attributes chosen for SSR (Search space reduction) are the bare minimum of what a system like this must have. Holding the assumption that there can be no property or search criteria accepted in the system without geo location and _sensible defaults_ will be used while searching without price/beds input from the user, this algorithm can keep performing good in all combinations of inputs.
If there is a new attribute added to the domain objects(property/search criterion) the algorithm can be fine tuned, only if required, to catch up with new searchable attributes. In case the new attributes aren't searchable, no modification is required to the current algorithm.

#### How will this be actually implemented? (Not HLD)
The scope of this activity was to demo a feature like this, where everything this is being done in memory and things like Geo Grid and data are a simulation. 
To go about actual implementation of this feature, there are a few options to for each unit of the mechanism stated in this doc.  
**Geo Grid**  
There are quite a few Geo Indexing database available in the market including Redis Geo API. Also a recent GeoHashing technique can be leveraged to build customised Geo Grid.  
**Data arrangement**  
Concatenated indexes can be leveraged in database systems to store the data in sorted price range buckets and then by sorted bedCount buckets. Most RDBMs systems support this. 
NoSQL DBs can also be a candidate as it will drastically reduce server side data transformation before sending to the frontend.

### Questions? Suggestions? Feedback?
Reach out jskrnbindra@gmail.com

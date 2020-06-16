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
The algorithm should be finite (terminating) function which deffinately terminates with a set of rich matches. The set of matches can be an empty set indicating **no match found**.

To accomplish this we need to:
- search
- match

While matching can be implemented efficiently and has scope only for micro-optimisation **search** is the subtask which has major scope of effective optimisation.

### The approach
One of the best ways to do faster search is to reduce the search space.
Naively this can be done in O(N) where N is the total number of records in the system i.e. properties while searching with criterion and criteria while searching with property.

In order to do it faster the search should be narrowed down to a subset of the universal set of records hereafter referred to as **U**.

The entities of **U** are complex objects with various attributes (which might also evolve). Search space reduction in this type of data needs to happen at multiple levels so as to have minimum query time. The aim is to have as less records as possible for match calculation. 

Analysing the attributes of both the types Property and Search Criterion there are a few indexable candidates which can be beneficial in eliminating records which the query is not interested in. Few of the effective attributes which might contributes to faster search are:
- Price/Cost
- latitude and longitude
- Number of bed/bath rooms

Each of these bear a weightage out of which Price and geolocation have high weightage. These attributes also come with toleraces which might be useful in cases when there is not enough search results to display or in advanced search.

The records in **U** need to be indexed on, atleast, the price and geolocation attributes which aid faster queries and insertions. But the order in which these indexes are evaluated play a great role in the execution time of a query.


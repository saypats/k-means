# k-means
A simple C++ implementation of k-means clustering

##Overview and notes on use

This collection of templates implements <i>k</i>-means clustering. 

This implementation of k-means employs the following strategy:
1. Pick a population member at random, assign its value as the centroid of cluster 0.
2. For clusters 1 through k-1: select the member of the population that is *farthest* from all
clusters whose centroids have already been assigned (where farthest is the maximum of the sum of
euclidean distances from the candidate population element to each of the centroids).
3. Assign each element of the population to the nearest cluster, where *nearest* corresponds to minimum
euclidean distance.
4. Recalculate the centroids of all clusters, such that the centroid is the mean of all of its member vectors.
5. Repeat steps 3 and 4 until the cluster memberships are stable, that is, two consecutive iterations result
in the same cluster memberships.

A simple use case includes these steps:

1. Define the problem space, typically by instantiating the k_means class template
2. Prepare input data
3. Construct an instance of (previously instantiated) k_means template class, passing it the input data
4. Obtain the resulting cluster objects from the k_means

###Define the problem space

The problem space is defined by the *dimensional cardinality* of the space, and the underlying scalar type (e.g., `double`)
used to represent vector components. These correspond (in reverse order) to the template parameters for k_means<T,N>. 
For example, to define a 2-dimensional space with scalar elements of type `double`:
````C++
	using km_type = utils::k_means<double, 2>;
````
or, if you prefer:
````C++
	typedef utils::k_means<double, 2> km_type;
````

This alias (or `typedef`) makes it easer to define instances of related types, using aliases
defined in the k_means template, to wit:
+ `km_type::vector_type` : the type of a vector in the problem space, i.e., the type of input data
+ `km_type::scalar_type` : the underlying aritnmetic type of vector components (T template parameter)
+ `km_type::population_type` : the type of the container for input data
+ `km_type::cluster_type` : the type of clusters generated by the k-means algorithm, i.e., an element of the output
+ `km_type::cluster_container_type` : the type of the container of clusters

Note that the alias declaration does not, in itself, instantiate the class template. If you use one
of the types above in a definition, for example:
````C++
km_type::population_type data = { ... };
````
then the class template will be instantiated.

###Input data

####Vector type

Given the previous definition of `km_type`, the alias `km_type::vector_type`
corresponds to `utils::euclidean_vector<double,2>`. The `euclidean_vector` class template is used to represent
any N-dimensional value (where N > 1) in the problem space, including input data and cluster centroids.
The template is derived publicly from `std:array`, inheriting its interface. For the most part, programs 
using k-means will use instances of euclidean_vector just as if they were instances of std::array---they 
can be constructed with initializer lists:
````C++
	km_type::vector_type v{1.43, 0.95};
````
Individual components can be accessed or set with the subscript operator:
````C++
	double x = v[0];
	v[1] = x * x;
````

####Specialization for N=1

If the problem space is one-dimensional, partial template specializations redefine the component types of 
cluster and k_means to avoid using the euclidean_vector template for data values. Essentially, 
everywhere `euclidean_vector<T,N>` is used as a parameter, return type, or component type when N > 1,
it is replaced by the scalar type T when N = 1. For example:

+ `vector_type` is aliased to `T`, the scalar type
+ `population_type` is aliased to `std::vector<T>`
+ `cluster::centroid()` returns a value of type `T`

and so on.

####Input container

The type `population_type`
is an alias for `std::vector<km_type::vector_type>`, that is, a container of input data. The population
vector can be inialized in any number of ways, including the use of an initializer list:
````
km_type::population_type input =
{
	{0.0, 0.0},
	{0.2, -0.1},
	{1.0, 1.0},
	{1.2, 0.8},
	{-1.0, -1.0},
	{-1.1, -0.9}
};
````

####Population member identity

Each member of the input population is implicitly identified by its index in the population vector. The programmer
is responsible for maintaining an association between the application objects being clustered and their corresponding
vectors in the population. In particular, the output clusters describe their contents (that is, which 
data are members of the cluster) in terms of the members' indices in the population vector.

###Construct the k_means instance

The k_means constructor takes two parameters:
````
k_means(const population_type &population, std::size_t k)
````
The first, `population`, is the input data set. The second, `k`, is the number
of clusters into which the population will be partitioned.

Given the previous definitions of `km_type`, and the population vector `input`, we can construct
the result set:
````
	km_type km(input, 3);
````

####Determining a value for *k*

A value of k (the number of clusters into which the population is partitioned) must be specified.
In most cases, an optimal value for k is not known *a priori*. Strategies for determining a
reasonable value for k  are beyond the scope of this discussion, but many such strategies are
available from internet sources. Many strategies use the sum of squared errors of the population
as an objective function. This value is returned by `k_means::sse()`.


###Get the resulting clusters

The method k_means::clusters() returns a reference to the vector containing the resulting cluster objects.
When iterating over this vector, don't assume that the size of the vector is equal to k (specified in the constructor).
In some circumstances, the actual number of resulting clusters may be fewer than k. From each cluster, you can get
information about the cluster, such as the centroid and the standard deviation. 
The method cluster::members() returns a container of type `std::set<std::size_t>`.
This set contains the indices in the population vector of elements that belong to the cluster:
````
for (auto i = 0; i < km.clusters().size(); ++i)
{
	auto& cluster = km.clusters()[i];
	
	std::cout << "cluster " << i;
	std::cout << " - centroid: " << cluster.centroid();
	std::cout << ", sigma: " << cluster.sigma();
	std::cout << ", members: " << std::endl;
	for (auto it = cluster.members().cbegin(); it != cluster.members().cend(); ++it)
	{
		std::cout << "population[" << *it << "] : " << input[*it] << std::endl;
	}
}
````
Note that `euclidean_vector` provides a stream output operator (`<<`),
which prints the values in the vector enclosed in braces {}, separated by commas. The output 
of this loop should look like this (the cluster order may vary):
````
cluster 0 - centroid: { 1.1, 0.9 }, sigma: 0.141421, members:
population[2] : { 1, 1 }
population[3] : { 1.2, 0.8 }
cluster 1 - centroid: { -1.05, -0.95 }, sigma: 0.0707107, members: 
population[4] : { -1, -1 }
population[5] : { -1.1, -0.9 }
cluster 2 - centroid: { 0.1, -0.05 }, sigma: 0.111803, members: 
population[0] : { 0, 0 }
population[1] : { 0.2, -0.1 }
````

Alternatively, you can get a single *result vector* from the k_means instance:
````
auto& results = km.result_vector();
````
This type of this vector is `std::vector<std::size_t>`. It is, essentially, a map from population element indices
to cluster indices---the i<sup>th</sup> element of the result vector contains the index of the cluster (in the vector
returned by k_means::clusters()) to which the i<sup>th</sup> element of the population vector belongs:
````
auto& result_vec = km.result_vector();
for (auto i = 0; i < result_vec.size(); ++i)
{
	auto icluster = result_vec[i];
	std::cout << "input element " << i << " " << input[i] << " is a member of cluster " << icluster << std::endl;
}
````

The output of this loop should look like this (again, cluster indices/order may vary):
````
input element 0 { 0, 0 } is a member of cluster 2
input element 1 { 0.2, -0.1 } is a member of cluster 2
input element 2 { 1, 1 } is a member of cluster 0
input element 3 { 1.2, 0.8 } is a member of cluster 0
input element 4 { -1, -1 } is a member of cluster 1
input element 5 { -1.1, -0.9 } is a member of cluster 1
````


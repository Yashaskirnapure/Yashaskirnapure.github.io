## HyperLogLog - Estimating Cardinality of Large Datasets

When generating and managing large amounts of data, a useful metric to extract insights from is to find the cardinality of the set. For those who have no idea what these terms mean, cardinality is basically the number of unique entries in a particular dataset. As for sets, it could mean a lot of things depending on the context.

In mathematics, a set is a homogeneous structure that stores distinct values in an unordered fashion. Since they already maintain uniqueness, the cardinality is essentially the size of the set. But in the real world, data is rarely that tidy. Logs, tables and streams are full of noise and duplicates, so before we actually start talking about cardinality we need strip this data down to it distinct values. Throughout this article, when I say 'set', I’ll be referring to data in that looser, practical sense.

### Finding unique values

So, let us look at this problem from a programatic perspective. If we imagine our data as an array of numbers, the problem boils down to finding the number of unique items in the array. There are basically 2 ways we could tackle this-

#### 1. Brute Force
The most straightforward approach is to compare each element with every other element.  
- Time Complexity: O(n²)  
- Space Complexity: O(1)  

#### 2. Sorting + Scan
We can do better by first sorting the array. This brings equal items next to each other, so we only need to compare each element with its immediate predecessor.  
- Time Complexity: O(n log n) for sorting + O(n) for traversal = O(n log n)  
- Space Complexity: O(1) if using in-place sort  

#### 3. Hashing
Another approach is to use a hashmap to keep track of elements we’ve seen giving us the privelege of constant time lookups. This avoids sorting altogether.  
- Time Complexity: O(n) on average  
- Space Complexity: O(n)  


These approaches are fine, but taking into consideration the size of data we have, they may fall short. This is where HyperLogLog comes in.

### HyperLogLog - a probabilisic approach

All the methods we mentioned above are by nature deterministic. They provide exact steps of execution and will give a final distinct answer. When discussing HyperLogLog, we move from a deterministic approach to a probablistic approach where we try to estimate the cardinality instead of actually finding it.

The HyperLogLog tries to estimate the cardinality of the set by finding the rarest item in the group. This actually makes sense considering the assumption that a very rare item present in the group could very well mean there could be a lot of unique members in the group. Take for example a simple event like tossing 3 coins, what is the rarest event here ? either all heads or all tails. HyperLogLog extends this idea to estimate the cardinality of sets, but instead of finding the so called 'rarest' item, it hashes values and watches for patterns that are somewhat unlikely to appear in large datasets, specifically the maximum number of leading or trailing zeros in the hash value of the item. The longer/rarer the item the more unique elements you can infer.

#### From Probabilistic Counting to HyperLogLog

![Probabilistic counting](/_posts/assets/prob_counting.webp)

The idea of cardinality estimation started with **probabilistic counting**: hash the data and track the maximum run of leading zeros. While elegant, this single estimate was unstable.  

To improve accuracy, researchers proposed splitting the stream into multiple **estimators** (buckets). Each bucket ran its own tiny estimator, and their results were later aggregated. Finally, **HyperLogLog** refined this approach by combining bucket estimates using a harmonic mean and correcting systematic bias.

![Harmonic mean](/_posts/assets/harmonic_mean.webp)

Lets discuss the steps that involved in the algorithm -
1. **Hash each element**  
   Every incoming element is passed through a hash function to produce a uniformly distributed binary value. The actual element is never stored—only its hash.

2. **Split the hash**  
   The hash is divided into two parts:  
   - The first few bits decide which *bucket* (or register) this element belongs to.  
   - The remaining bits are used to detect patterns (leading zeros).

3. **Count leading zeros**  
   For the second part of the hash, we calculate how many leading zeros appear before the first 1. This count indicates how “rare” the pattern is.

4. **Update the register**  
   For that bucket, we keep track of the maximum number of leading zeros seen so far. Each bucket therefore records a tiny summary of the data it has observed.

![Hashing illustration](/_posts/assets/hll_hashing.webp)

After we are done with the generating and populating the registers, we aggregate the observations of each of the buckets. HyperLogLog proposes that we use the **Harmonic Mean** for this, owing to its ability to handle outliers without making the results unstable. In order to further improve the accuracy of the estimates, the algorithm multiplies the mean with a carefully calculated constant.

![Cardinality](/_posts/assets/final.png)

While the algorithm only provides an estimate of the cardinality of the data instead of an actual value, it achieves an error of ~2% which is actually a pretty good trade-off especially considering the size of data we are managing and the amount of compute and storage it might require to calculate exact values.
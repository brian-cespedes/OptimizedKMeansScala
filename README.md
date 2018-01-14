# OptimizedKMeansScala
Improving performance of K-Means on Spark. 

V2: Vanilla Spark MLlib K-Means.  
V4: Center Update K-Means.  
V5: YinYang K-Means.  


## Implementation   
 
  ### MLlib K-Means:      
   The K-means in this library is implemented using a 
     [scalable version of K-Means++](http://theory.stanford.edu/~sergei/papers/vldb12-kmpar.pdf) as shown below :  
             
       
   ![screen shot 2018-01-14 at 4 45 07 am](https://user-images.githubusercontent.com/15849566/34916035-c1420a2a-f8e5-11e7-876d-eed884d1b1cc.png)
   

  ### Design workflow :    
  <img width="786" alt="screen shot 2018-01-14 at 5 01 50 am" src="https://user-images.githubusercontent.com/15849566/34916153-141b2e82-f8e8-11e7-846a-1662f172d431.png">  
          
                          
   * Initially data is stored as RDD and centers are initialized using user specified method.  
   * Centers are broadcasted to workers and mapPartitions method called on the RDD.  
      * In mapPartitions, closest center computed for each point and partial (sum, count) computed by each worker.  
      * Each worker returns iterator containing partial sum and count for each center (k values) based on points in their 
        partitons.    
   * Reduce step then computes sum across all iterators for the k cluster centers. 
   * Center update simply scales the total sum by total number of points assigned to that cluster.  
   * The updated centers are used for the next iteration and repeated until a user specified number of iterations.    
   
  ### Optimizations :          
   * *Update* :  
     * Update step of K-Means is optimized using   
     
     <p align="center"> 
     <img width="500" alt="screen shot 2018-01-14 at 5 36 56 am" src="https://user-images.githubusercontent.com/15849566/34916423-f9d02000-f8ec-11e7-8630-9a626d28707a.png">
     </p>

     * Herer c' is newClusterCenter, c is oldClusterCenter, V' is set of points assigned to cluster in current iteration, V is 
       set of points assigned to cluster in previous iteration and OV is intersection between V' and V.    
     * New center update subtracts the points that have left the cluster and adds the points that have been included in the 
       current iteration from the total sum of previous iteration, followed by dividing by new cluster size.     
     * This improves perfromance since most points do not change cluster assignment after few iterations leading to 
       fewer computations on each worker machine. 
     * There is an overhead of storing the cluster assignment for each data point as we transformed the initial dataset RDD into 
       a new RDD that consists of a (point, cluster assignment) pair where the initial cluster assignment for all points is set 
       to -1.  
     * In each iteration a map function is applied on the RDD to update cluster assignment. 
     * As shown below in the code snippet the part on the left shows the check done to see if the value of bestCenter for a given 
       point is the same in current iteration as that in the previous iteration .If it turns out not to be the same, we 
       are aware that clusterAssignment has changed with newClusterAssingment represented by bestCenter and oldClusterAssignment 
       represented by point._2 .
       
     * Provided clusterAssignment has changed, the point is added to partial sum for new center and subtracted from old center.
     * The code snippet on right shows center update function that involves adding the previous sum of points for a cluster like 
       shown in the equation above.    
   
  <p align="center"> 
  <img width="5151" alt="screen shot 2018-01-14 at 10 00 45 am" src="https://user-images.githubusercontent.com/15849566/34919010-d8496dc2-f911-11e7-8b8a-4e88e3f9e2ad.png">
  </p>

   * *Yinyang* :
      * This approach optimizes the cluster assignment step with reference to a paper in ICML named 
      [Yinyang K-Means](https://people.engr.ncsu.edu/xshen5/Publications/icml15.pdf).      
      * The idea is to use upper and lower bounds to reduce number of distance computations. It makes use of triangle inequality 
      to state that for any points a,b,c and using Eucledian distance
      <p align="center"> 
      <img width="250" alt="screen shot 2018-01-14 at 10 33 09 am" src="https://user-images.githubusercontent.com/15849566/34919305-659193a4-f916-11e7-9b1f-56c1af2f11e0.png"> 
      </p>
      
      * The paper uses a concept of *global filtering* to determine whether a point changed its cluster assignment. A point x did 
      not change its cluster if 
      
      <p align = "center">
      <img width="350" alt="screen shot 2018-01-14 at 10 44 44 am" src="https://user-images.githubusercontent.com/15849566/34919389-f9c0c6fc-f917-11e7-874e-038fe9a2ecb1.png">
      </p>
       
       * b(x) is closest center assigned to point x, ub(x) is upper bound of point x, lb(x) is lower bound of point x and 
         delta(b) is change in distance of center b.  
       * The paper modifies the global filtering and introduces two types of filtering group and local. Only group filtering is 
         implemented here that updates the various bounds.
       * The implementation of YinYang algorithm starts with transfroming each point to a RDD of tuples represented as (point, 
         clusterAssignment, upperBound, lowerBound). Center update is used as described above to optimize the update step.
       * Instead of using local filtering, standard distance computation to each center was used for such points.    
       
       <p align = "center">
       <img width="1014" alt="screen shot 2018-01-14 at 10 53 23 am" src="https://user-images.githubusercontent.com/15849566/34919555-b2b4ce18-f91a-11e7-8f8c-88bc09b58739.png">
       </p>
       
   ### Experiments:
   * Datasets were generated using make_blobs function in scikit-learn in python. Varying value of n_samples(n) and 
         n_features(d) allowed us to get dataset size varying from 250MB to 4GB as shown below
       
       <p align = "center">
       <img width="739" alt="screen shot 2018-01-14 at 11 02 31 am" src="https://user-images.githubusercontent.com/15849566/34919538-7b930d8c-f91a-11e7-8463-8264c3ee0ebf.png">
        </p>
        
   * Amazon EC2 instances were setup using a Large Amazon Machine Image (6.5 ECUs, 2 vCPUs, 2.4 GHz, Intel Xeon E5-2676v3, 8 
         GiB memory, EBS only) assigning the number of instances to be three.  
   * Then a key-pair was created, Amazon security credentials set and the EC2 instances launched.  
   * After generating the datasets, a master and 2 slaves were initialized and in order to run our version of K-Means a spark 
         job was submitted to the master.       

        <p align = "center">
         <img width="2500" alt="screen shot 2018-01-14 at 11 16 32 am" src="https://user-images.githubusercontent.com/15849566/34919660-731b8cae-f91c-11e7-8755-5cb0ee2ee20b.png">
        </p>
         
   ### Results:
       
   <p align = "center">
       <img width="999" alt="screen shot 2018-01-14 at 11 18 44 am" src="https://user-images.githubusercontent.com/15849566/34919720-c0187b0c-f91c-11e7-97d9-cd667f0fc52e.png">
   </p>
       
   ### Conclusion:
   * Center update and Yinyang optimization perform better(lower time) than vanilla K-Means of MLlib Spark until datasets of size 
   4GB.
   * After that MLlib K-Means is slightly faster than Yinyang and Yinyang faster than center update version.
   * Implemented two different versions of K-Means and tested them on small and medium sized datasets.
   * Deployed the implemented algorithms on cluster in Amazon EC2.
   * Average speed up acheived is 1.98x for center update K-Means and 1.6x for Yinyang K-Means.    
   
   ### References:
   * [Spark:Cluster Computing with Working Sets](https://people.csail.mit.edu/matei/papers/2010/hotcloud_spark.pdf).
   * [Yinyang K-Means](https://people.engr.ncsu.edu/xshen5/Publications/icml15.pdf)
   * [Resilient distributed datasets: A fault-tolerant abstraction for in-memory cluster computing](https://www.usenix.org/system/files/conference/nsdi12/nsdi12-final138.pdf)
   * [EC2 setup for Spark](http://spark.apache.org/docs/latest/ec2-scripts.html)
   * [Spark MLlib](http://spark.apache.org/docs/latest/mllib-guide.html)
   

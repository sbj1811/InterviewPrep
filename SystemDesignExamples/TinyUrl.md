
## Designing a URL Shortening service like TinyURL

Let’s design a URL shortening service like TinyURL. This service will provide short aliases redirecting to long URLs. Similar services: bit.ly, goo.gl,  [qlink.me  7](http://qlink.me/), etc. Difficulty Level: Easy

1.  Why do we need URL shortening?

URL shortening is used to create shorter aliases for long URLs. We call these shortened aliases “short links.” Users are redirected to the original URL when they hit these short links. Short links save a lot of space when displayed, printed, messaged or tweeted. Additionally, users are less likely to mistype shorter URLs.

For example, if we shorten this page through TinyURL:

> [https://www.educative.io/collection/page/5668639101419520/5649050225344512/5668600916475904/  26](https://www.educative.io/collection/page/5668639101419520/5649050225344512/5668600916475904/)

We would get:

> [http://tinyurl.com/jlg8zpc  23](http://tinyurl.com/jlg8zpc)

The shortened URL is nearly one-third of the size of the actual URL.

URL shortening is used for optimizing links across devices, tracking individual links to analyze audience and campaign performance, and hiding affiliated original URLs.

If you haven’t used  [tinyurl.com  18](http://tinyurl.com/)  before, please try creating a new shortened URL and spend some time going through the various options their service offers. This will help you a lot in understanding this chapter better.

2.  Requirements and Goals of the System

![:bulb:](https://coursehunters.online/images/emoji/facebook_messenger/bulb.png?v=9 ":bulb:")  You should always clarify requirements at the beginning of the interview. Be sure to ask questions to find the exact scope of the system that the interviewer has in mind.

Our URL shortening system should meet the following requirements:

**Functional Requirements:**

1.  Given a URL, our service should generate a shorter and unique alias of it. This is called a short link.
2.  When users access a short link, our service should redirect them to the original link.
3.  Users should optionally be able to pick a custom short link for their URL.
4.  Links will expire after a standard default timespan. Users should also be able to specify the expiration time.

**Non-Functional Requirements:**

1.  The system should be highly available. This is required because, if our service is down, all the URL redirections will start failing.
2.  URL redirection should happen in real-time with minimal latency.
3.  Shortened links should not be guessable (not predictable).

**Extended Requirements:**

1.  Analytics; e.g., how many times a redirection happened?
2.  Our service should also be accessible through REST APIs by other services.

3.  Capacity Estimation and Constraints

Our system will be read-heavy. There will be lots of redirection requests compared to new URL shortenings. Let’s assume 100:1 ratio between read and write.

**Traffic estimates:**  If we assume we will have 500M new URL shortenings per month, we can expect (100 * 500M => 50B) redirections during that same period. What would be Queries Per Second (QPS) for our system?

New URLs shortenings per second:

500 million / (30 days * 24 hours * 3600 seconds) = ~200 URLs/s

URLs redirections per second, considering 100:1 read/write ratio:

50 billion / (30 days * 24 hours * 3600 sec) = ~19K/s

**Storage estimates:**  Let’s assume we store every URL shortening request (and associated shortened link) for 5 years. Since we expect to have 500M new URLs every month, the total number of objects we expect to store will be 30 billion:

500 million * 5 years * 12 months = 30 billion

Let’s assume that each stored object will be approximately 500 bytes (just a ballpark estimate–we will dig into it later). We will need 15TB of total storage:

**Bandwidth estimates:**  For write requests, since we expect 200 new URLs every second, total incoming data for our service will be 100KB per second:

200 * 500 bytes = 100 KB/s

For read requests, since every second we expect ~19K URLs redirections, total outgoing data for our service would be 9MB per second.

19K * 500 bytes = ~9 MB/s

**Memory estimates:**  If we want to cache some of the hot URLs that are frequently accessed, how much memory will we need to store them? If we follow the 80-20 rule, meaning 20% of URLs generate 80% of traffic, we would like to cache these 20% hot URLs.

Since we have 19K requests per second, we will be getting 1.7 billion requests per day:

19K * 3600 seconds * 24 hours = ~1.7 billion

To cache 20% of these requests, we will need 170GB of memory.

0.2 * 1.7 billion * 500 bytes = ~170GB

**High level estimates:**  Assuming 500 million new URLs per month and 100:1 read:write ratio, following is the summary of the high level estimates for our service:

	New URLs200/s

	URL redirections 19K/s

	Incoming data 100KB/s

	Outgoing data 9MB/s

	Storage for 5 years 15TB

	Memory for cache 170GB

4.  System APIs

![:bulb:](https://coursehunters.online/images/emoji/facebook_messenger/bulb.png?v=9 ":bulb:")  Once we’ve finalized the requirements, it’s always a good idea to define the system APIs. This should explicitly state what is expected from the system.

We can have SOAP or REST APIs to expose the functionality of our service. Following could be the definitions of the APIs for creating and deleting URLs:

creatURL(api_dev_key, original_url, custom_alias=None, user_name=None, expire_date=None)

**Parameters:**  
api_dev_key (string): The API developer key of a registered account. This will be used to, among other things, throttle users based on their allocated quota.  
original_url (string): Original URL to be shortened.  
custom_alias (string): Optional custom key for the URL.  
user_name (string): Optional user name to be used in encoding.  
expire_date (string): Optional expiration date for the shortened URL.

**Returns:**  (string)  
A successful insertion returns the shortened URL; otherwise, it returns an error code.

deleteURL(api_dev_key, url_key)

Where “url_key” is a string representing the shortened URL to be retrieved. A successful deletion returns ‘URL Removed’.

**How do we detect and prevent abuse?**  A malicious user can put us out of business by consuming all URL keys in the current design. To prevent abuse, we can limit users via their api_dev_key. Each api_dev_key can be limited to a certain number of URL creations and redirections per some time period (which may be set to a different duration per developer key).

5.  Database Design

![:bulb:](https://coursehunters.online/images/emoji/facebook_messenger/bulb.png?v=9 ":bulb:")  Defining the DB schema in the early stages of the interview would help to understand the data flow among various components and later would guide towards the data partitioning.A few observations about the nature of the data we will store:

1.  We need to store billions of records.
2.  Each object we store is small (less than 1K).
3.  There are no relationships between records—other than storing which user created a URL.
4.  Our service is read-heavy.

#### Database Schema:

We would need two tables: one for storing information about the URL mappings, and one for the user’s data who created the short link.

![](https://www.educative.io/api/collection/5668639101419520/5649050225344512/page/5668600916475904/image/5685265389584384.png)![](data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==)

**What kind of database should we use?**  Since we anticipate storing billions of rows, and we don’t need to use relationships between objects – a NoSQL key-value store like Dynamo or Cassandra is a better choice. A NoSQL choice would also be easier to scale. Please see  [SQL vs NoSQL  52](https://www.educative.io/collection/page/5668639101419520/5649050225344512/5728116278296576/)  for more details.

6.  Basic System Design and Algorithm

The problem we are solving here is: how to generate a short and unique key for a given URL?

In the TinyURL example in Section 1, the shortened URL is “[http://tinyurl.com/jlg8zpc”  6](http://tinyurl.com/jlg8zpc%E2%80%9D). The last six characters of this URL is the short key we want to generate. We’ll explore two solutions here:

### a. Encoding actual URL

We can compute a unique hash (e.g.,  [MD5  5](https://en.wikipedia.org/wiki/MD5)  or  [SHA256  6](https://en.wikipedia.org/wiki/SHA-2), etc.) of the given URL. The hash can then be encoded for displaying. This encoding could be base36 ([a-z ,0-9]) or base62 ([A-Z, a-z, 0-9]) and if we add ‘-’ and ‘.’, we can use base64 encoding. A reasonable question would be: what should be the length of the short key? 6, 8 or 10 characters?

Using base64 encoding, a 6 letter long key would result in 64^6 = ~68.7 billion possible strings  
Using base64 encoding, an 8 letter long key would result in 64^8 = ~281 trillion possible strings

With 68.7B unique strings, let’s assume for our system six letter keys would suffice.

If we use the MD5 algorithm as our hash function, it’ll produce a 128-bit hash value. After base64 encoding, we’ll get a string having more than 21 characters (since each base64 character encodes 6 bits of the hash value). Since we only have space for 8 characters per short key, how will we choose our key then? We can take the first 6 (or 8) letters for the key. This could result in key duplication though, upon which we can choose some other characters out of the encoding string or swap some characters.

**What are different issues with our solution?**  We have the following couple of problems with our encoding scheme:

1.  If multiple users enter the same URL, they can get the same shortened URL, which is not acceptable.
2.  What if parts of the URL are URL-encoded? e.g.,  [http://www.educative.io/distributed.php?id=design  4](http://www.educative.io/distributed.php?id=design), and  [http://www.educative.io/distributed.php%3Fid%3Ddesign  3](http://www.educative.io/distributed.php%3Fid%3Ddesign)  are identical except for the URL encoding.

**Workaround for the issues:**  We can append an increasing sequence number to each input URL to make it unique, and then generate a hash of it. We don’t need to store this sequence number in the databases, though. Possible problems with this approach could be an ever-increasing sequence number. Can it overflow? Appending an increasing sequence number will also impact the performance of the service.

Another solution could be to append user id (which should be unique) to the input URL. However, if the user has not signed in, we would have to ask the user to choose a uniqueness key. Even after this, if we have a conflict, we have to keep generating a key until we get a unique one.  

[![1](https://coursehunters.online/uploads/default/optimized/1X/e7d3ce09ceeb3a5dd3c8ed5ed8fca01c5984f35f_2_690x247.png)](https://coursehunters.online/uploads/default/original/1X/e7d3ce09ceeb3a5dd3c8ed5ed8fca01c5984f35f.png "1.png")

1.png1006×361 38.1 KB

[![2](https://coursehunters.online/uploads/default/optimized/1X/035279c758eff8abb91aea1007bfd439489faacf_2_690x345.png)](https://coursehunters.online/uploads/default/original/1X/035279c758eff8abb91aea1007bfd439489faacf.png "2.png")

2.png707×354 11 KB

![3](https://coursehunters.online/uploads/default/original/1X/101baa23656a54ab27578adbb60331d2d58ddb54.png)

[![4](https://coursehunters.online/uploads/default/optimized/1X/a86c5bb92765821bc14a551ab3fe85f802172024_2_690x204.png)](https://coursehunters.online/uploads/default/original/1X/a86c5bb92765821bc14a551ab3fe85f802172024.png "4.png")

4.png1190×353 21.5 KB

[![5](https://coursehunters.online/uploads/default/optimized/1X/f99c0f43474d6750acb447a9fee03343cfa6d858_2_690x217.png)](https://coursehunters.online/uploads/default/original/1X/f99c0f43474d6750acb447a9fee03343cfa6d858.png "5.png")

5.png1214×382 26.5 KB

[![6](https://coursehunters.online/uploads/default/optimized/1X/d9e66ba36bd542ad56e523911904821147e94f34_2_690x215.png)](https://coursehunters.online/uploads/default/original/1X/d9e66ba36bd542ad56e523911904821147e94f34.png "6.png")

6.png1198×374 31.5 KB

[![7](https://coursehunters.online/uploads/default/optimized/1X/4278bc99645323e3506a4fd309854d0f239c2230_2_689x218.png)](https://coursehunters.online/uploads/default/original/1X/4278bc99645323e3506a4fd309854d0f239c2230.png "7.png")

7.png1212×383 31.5 KB

[![8](https://coursehunters.online/uploads/default/optimized/1X/ecbe01b84d4440716eab1ea410ac5a6344449424_2_690x212.png)](https://coursehunters.online/uploads/default/original/1X/ecbe01b84d4440716eab1ea410ac5a6344449424.png "8.png")

8.png1202×370 35.2 KB

[![9](https://coursehunters.online/uploads/default/optimized/1X/a04769d1e94881225777a3e12e4b4b0bee4908f4_2_690x217.png)](https://coursehunters.online/uploads/default/original/1X/a04769d1e94881225777a3e12e4b4b0bee4908f4.png "9.png")

9.png1185×374 39.6 KB

### b. Generating keys offline

We can have a standalone Key Generation Service (KGS) that generates random six letter strings beforehand and stores them in a database (let’s call it key-DB). Whenever we want to shorten a URL, we will just take one of the already-generated keys and use it. This approach will make things quite simple and fast. Not only are we not encoding the URL, but we won’t have to worry about duplications or collisions. KGS will make sure all the keys inserted into key-DB are unique

**Can concurrency cause problems?**  As soon as a key is used, it should be marked in the database to ensure it doesn’t get used again. If there are multiple servers reading keys concurrently, we might get a scenario where two or more servers try to read the same key from the database. How can we solve this concurrency problem?

Servers can use KGS to read/mark keys in the database. KGS can use two tables to store keys: one for keys that are not used yet, and one for all the used keys. As soon as KGS gives keys to one of the servers, it can move them to the used keys table. KGS can always keep some keys in memory so that it can quickly provide them whenever a server needs them.

For simplicity, as soon as KGS loads some keys in memory, it can move them to the used keys table. This ensures each server gets unique keys. If KGS dies before assigning all the loaded keys to some server, we will be wasting those keys–which is acceptable, given the huge number of keys we have.

KGS also has to make sure not to give the same key to multiple servers. For that, it must synchronize (or get a lock to) the data structure holding the keys before removing keys from it and giving them to a server

**What would be the key-DB size?**  With base64 encoding, we can generate 68.7B unique six letters keys. If we need one byte to store one alpha-numeric character, we can store all these keys in:

6 (characters per key) * 68.7B (unique keys) = 412 GB.

**Isn’t KGS the single point of failure?**  Yes, it is. To solve this, we can have a standby replica of KGS. Whenever the primary server dies, the standby server can take over to generate and provide keys.

**Can each app server cache some keys from key-DB?**  Yes, this can surely speed things up. Although in this case, if the application server dies before consuming all the keys, we will end up losing those keys. This could be acceptable since we have 68B unique six letter keys.

**How would we perform a key lookup?**  We can look up the key in our database or key-value store to get the full URL. If it’s present, issue an “HTTP 302 Redirect” status back to the browser, passing the stored URL in the “Location” field of the request. If that key is not present in our system, issue an “HTTP 404 Not Found” status, or redirect the user back to the homepage.

**Should we impose size limits on custom aliases?**  Our service supports custom aliases. Users can pick any ‘key’ they like, but providing a custom alias is not mandatory. However, it is reasonable (and often desirable) to impose a size limit on a custom alias to ensure we have a consistent URL database. Let’s assume users can specify a maximum of 16 characters per customer key (as reflected in the above database schema).

![No Graph available](data:image/png;base64,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)

High level system design for URL shortening

7.  Data Partitioning and Replication

To scale out our DB, we need to partition it so that it can store information about billions of URLs. We need to come up with a partitioning scheme that would divide and store our data to different DB servers.

**a. Range Based Partitioning:**  We can store URLs in separate partitions based on the first letter of the URL or the hash key. Hence we save all the URLs starting with letter ‘A’ in one partition, save those that start with letter ‘B’ in another partition and so on. This approach is called range-based partitioning. We can even combine certain less frequently occurring letters into one database partition. We should come up with a static partitioning scheme so that we can always store/find a file in a predictable manner.

The main problem with this approach is that it can lead to unbalanced servers. For example: we decide to put all URLs starting with letter ‘E’ into a DB partition, but later we realize that we have too many URLs that start with letter ‘E’.

**b. Hash-Based Partitioning:**  In this scheme, we take a hash of the object we are storing. We then calculate which partition to use based upon the hash. In our case, we can take the hash of the ‘key’ or the actual URL to determine the partition in which we store the data object.

Our hashing function will randomly distribute URLs into different partitions (e.g., our hashing function can always map any key to a number between [1…256]), and this number would represent the partition in which we store our object.

This approach can still lead to overloaded partitions, which can be solved by using  [Consistent Hashing  24](https://www.educative.io/collection/page/5668639101419520/5649050225344512/5709068098338816/).

8.  Cache

We can cache URLs that are frequently accessed. We can use some off-the-shelf solution like Memcache, which can store full URLs with their respective hashes. The application servers, before hitting backend storage, can quickly check if the cache has the desired URL.

**How much cache should we have?**  We can start with 20% of daily traffic and, based on clients’ usage pattern, we can adjust how many cache servers we need. As estimated above, we need 170GB memory to cache 20% of daily traffic. Since a modern day server can have 256GB memory, we can easily fit all the cache into one machine. Alternatively, we can use a couple of smaller servers to store all these hot URLs.

**Which cache eviction policy would best fit our needs?**  When the cache is full, and we want to replace a link with a newer/hotter URL, how would we choose? Least Recently Used (LRU) can be a reasonable policy for our system. Under this policy, we discard the least recently used URL first. We can use a  [Linked Hash Map  9](https://docs.oracle.com/javase/7/docs/api/java/util/LinkedHashMap.html)  or a similar data structure to store our URLs and Hashes, which will also keep track of which URLs are accessed recently.

To further increase the efficiency, we can replicate our caching servers to distribute load between them.

**How can each cache replica be updated?**  Whenever there is a cache miss, our servers would be hitting a backend database. Whenever this happens, we can update the cache and pass the new entry to all the cache replicas. Each replica can update their cache by adding the new entry. If a replica already has that entry, it can simply ignore it.  

[![6](https://coursehunters.online/uploads/default/optimized/1X/e40cf740e7b1ba6d00cd72d257729cc639c06655_2_690x360.png)](https://coursehunters.online/uploads/default/original/1X/e40cf740e7b1ba6d00cd72d257729cc639c06655.png "6.png")

6.png883×461 29.1 KB

9.  Load Balancer (LB)

We can add a Load balancing layer at three places in our system:

1.  Between Clients and Application servers
2.  Between Application Servers and database servers
3.  Between Application Servers and Cache servers

Initially, we could use a simple Round Robin approach that distributes incoming requests equally among backend servers. This LB is simple to implement and does not introduce any overhead. Another benefit of this approach is, if a server is dead, LB will take it out of the rotation and will stop sending any traffic to it.

A problem with Round Robin LB is that server load is not taken into consideration. If a server is overloaded or slow, the LB will not stop sending new requests to that server. To handle this, a more intelligent LB solution can be placed that periodically queries the backend server about its load and adjusts traffic based on that.

10.  Purging or DB cleanup

Should entries stick around forever or should they be purged? If a user-specified expiration time is reached, what should happen to the link?

If we chose to actively search for expired links to remove them, it would put a lot of pressure on our database. Instead, we can slowly remove expired links and do a lazy cleanup. Our service will make sure that only expired links will be deleted, although some expired links can live longer but will never be returned to users.

-   Whenever a user tries to access an expired link, we can delete the link and return an error to the user.
-   A separate Cleanup service can run periodically to remove expired links from our storage and cache. This service should be very lightweight and can be scheduled to run only when the user traffic is expected to be low.
-   We can have a default expiration time for each link (e.g., two years).
-   After removing an expired link, we can put the key back in the key-DB to be reused.
-   Should we remove links that haven’t been visited in some length of time, say six months? This could be tricky. Since storage is getting cheap, we can decide to keep links forever.

[![%D0%B7%D0%B0%D0%B2%D0%B0%D0%BD%D1%82%D0%B0%D0%B6%D0%B5%D0%BD%D0%BD%D1%8F%20(1)](https://coursehunters.online/uploads/default/optimized/1X/5f04cbfc6ae09fa15e32cf02a4307d8bf08ef7ce_2_690x323.png)](https://coursehunters.online/uploads/default/original/1X/5f04cbfc6ae09fa15e32cf02a4307d8bf08ef7ce.png "завантаження (1).png")

завантаження (1).png849×398 29.4 KB

11.  Telemetry

How many times a short URL has been used, what were user locations, etc.? How would we store these statistics? If it is part of a DB row that gets updated on each view, what will happen when a popular URL is slammed with a large number of concurrent requests?

Some statistics worth tracking: country of the visitor, date and time of access, web page that refers the click, browser or platform from where the page was accessed.

12.  Security and Permissions

Can users create private URLs or allow a particular set of users to access a URL?

We can store permission level (public/private) with each URL in the database. We can also create a separate table to store UserIDs that have permission to see a specific URL. If a user does not have permission and tries to access a URL, we can send an error (HTTP 401) back. Given that we are storing our data in a NoSQL wide-column database like Cassandra, the key for the table storing permissions would be the ‘Hash’ (or the KGS generated ‘key’). The columns will store the UserIDs of those users that have permissions to see the URL.
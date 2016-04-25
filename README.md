# cs-application-crawling-lab

## Learning goals

1.  Analyze the performance of Web indexing algorithms.
2.  Implement a Web crawler.


## Overview

In this lab, we present our solution to the previous lab and analyze the performance of Web indexing algorithms.  Then we build a simple Web crawler.


## Our Redis-backed indexer

In our solution, we store two kinds of structures in Redis:

*  For each search term, we have a `URLSet`, which is a Redis Set of URLs that contain the search term.

*  For each URL, we have a `TermCounter`, which is a Redis Hash that maps each search term to the number of times it appears.

We discussed these data types in the previous lab.  You can also [read about Redis Sets and Hashes here](http://redis.io/topics/data-types).

In `JedisIndex`, we provide a function that takes a search term and returns the Redis key of its URLSet:

```java
private String urlSetKey(String term) {
    return "URLSet:" + term;
}
```

And a function that takes a URL and returns the Redis key of its `TermCounter`:

```java
private String termCounterKey(String url) {
    return "TermCounter:" + url;
}
```

Here's our implementation of `indexPage`, which takes a URL and a JSoup `Elements` object that contains the DOM tree of the paragraphs we want to index:

```java
public void indexPage(String url, Elements paragraphs) {
    System.out.println("Indexing " + url);

    // make a TermCounter and count the terms in the paragraphs
    TermCounter tc = new TermCounter(url);
    tc.processElements(paragraphs);

    // push the contents of the TermCounter to Redis
    pushTermCounterToRedis(tc);
}
```

To index a page, we

1.   Make a Java `TermCounter` for the contents of the page, using code from a previous lab.

2.   Push the contents of the `TermCounter` to Redis.

Here's the new code that pushes a `TermCounter` to Redis:

```java
public List<Object> pushTermCounterToRedis(TermCounter tc) {
    Transaction t = jedis.multi();

    String url = tc.getLabel();
    String hashname = termCounterKey(url);

    // if this page has already been indexed; delete the old hash
    t.del(hashname);

    // for each term, add an entry in the termcounter and a new
    // member of the index
    for (String term: tc.keySet()) {
        Integer count = tc.get(term);
        t.hset(hashname, term, count.toString());
        t.sadd(urlSetKey(term), url);
    }
    List<Object> res = t.exec();
    return res;
}
```

This method uses a `Transaction` to collect the operations and send them to the server all at once, which is much faster than sending a series of small operations.

It loops through the terms in the `TermCounter`.  For each one it

1.  Finds or creates a `TermCounter` on Redis, then adds a field for the new term.

2.  Finds or creates a `URLSet` on Redis, then adds the current URL.

If the page has already been indexed, we delete its old `TermCounter` before pushing the new contents.

That's it for indexing new pages.

The second part of the lab asked you to write `getCounts`, which takes a search term and returns a map from each URL where the term appears to the number of times it appears there.  Here is our solution:

```java
	public Map<String, Integer> getCounts(String term) {
		Map<String, Integer> map = new HashMap<String, Integer>();
		Set<String> urls = getURLs(term);
		for (String url: urls) {
			Integer count = getCount(url, term);
			map.put(url, count);
		}
		return map;
	}
```

This method uses two helpers:

*  `getURLs` takes a search term and returns the Set of URLs where the term appears.

*  `getCount` takes a URL and a term and returns the number of times the term appears at the given URL.

Here are the implementations:

```java
	public Set<String> getURLs(String term) {
		Set<String> set = jedis.smembers(urlSetKey(term));
		return set;
	}

	public Integer getCount(String url, String term) {
		String redisKey = termCounterKey(url);
		String count = jedis.hget(redisKey, term);
		return new Integer(count);
	}
```

Because of the way we designed the index, these methods are simple and efficient.


## Analysis of lookup

Suppose we have indexed `N` pages and discovered `M` unique seach terms.  How long will it take to look up a search term?  Think about your answer before you continue.

To look up a search term, we run `getCounts`, which

1.  Creates a map.

2.  Runs `getURLs` to get a Set of URLs.

3.  For each URL in the Set, it runs `getCount` and adds an entry to a HashMap.

`getURLs` takes time proportional to the number of URLs that contain the search term.  For rare terms, that might be a small number, but for common terms it might be as large as `N`.

Inside the loop, we run `getCount`, which finds a `TermCounter` on Redis, looks up a term, and adds an entry to a HashMap.  Those are all constant time operations, so the overall complexity of `getCounts` is O(N) in the worst case.  However, in practice the runtime is proportional to the number of pages that contain the term, which is normally much less than `N`.

This algorithm is about as efficient as it can be, in terms of algorithmic complexity, but it is very slow because it sends many small operations to Redis.  You can make it much faster using a `Transaction`.  You might want to do that as an exercise, or you can see our solution in `RedisIndex.java`.

## Analysis of indexing

Using the data structures we designed, how long will it take to index a page?  Again, think about your answer before you continue.

To index a page, we traverse its DOM tree, find all the `TextNode` objects, and split up the strings into search terms.  That all takes time proportional to the number of words on the page.

For each term, we increment a counter in a HashMap, which is a constant time operation.  So making the `TermCounter` takes time proportional to the number of words on the page.

Pushing the `TermCounter` to Redis requires deleting a `TermCounter`, which is linear in the number of unique terms. Then for each term we have to

1.  Add an element to a `URLSet`, and

2.  Add an element to a Redis `TermCounter`.

Both of these are constant time operations, so the total time to push the `TermCounter` is linear in the number of unique search terms.

In summary, making the `TermCounter` is proportional to the number of words on the page.  Pushing the `TermCounter` to Redis is proportional to the number of unique terms.

Since the number of words on the page usually exceeds the number of unique search terms, the overall complexity is proportional to the number of words on the page.  In theory a page might contain all search terms in the index, so the worst case performance is O(M), but we don't expect to see the worse case in practice.

This analysis suggests a way to improve performance: we should probably avoid indexing very common words.  First of all, they take up a lot of time and space, because they appear in almost every `URLSet` and `TermCounter`.  Furthermore, they are not very useful because they don't help identify relevant pages.

Most search engines avoid indexing common words, which are known in this context as [stop words](https://en.wikipedia.org/wiki/Stop_words).



## Graph traversal

If you did the "Getting to Philosophy" lab, you already have a program that reads a Wikipedia page, finds the first link, uses the link to load the next page, and repeats.  This program is a specialized kind of crawler, but when people say "Web crawler" they usually mean a program that:

*   Loads a starting page and indexes the contents,

*   Finds all the links on the page and adds the linked URLs to a queue, and

*   Works its way through the queue, loading pages, indexing them, and adding new URLs to the queue.

*   If it finds a URL in the queue that has already been indexed, it skips it.

You can think of the Web as a [graph](https://en.wikipedia.org/wiki/Graph_(discrete_mathematics)) where each page is a node and each link is a directed edge from one node to another.  Starting from a source node, a crawler traverses this graph, visiting each reachable node once.

The behavior of the queue determines what kind of traversal the crawler performs:

*   If the queue is first-in-first-out (FIFO), the crawler performs a breadth-first traversal.

*   If the queue is last-in-first-out (LIFO), the crawler performs a depth-first traversal.

*   More generally, the items in the queue might be prioritized.  For example, we might want to give higher priority to pages that have not been indexed for a long time.

You can [read more about graph traversal here](https://en.wikipedia.org/wiki/Graph_traversal)




## Making a crawler

Now it's time to write the crawler.

When you check out the repository for this lab, you should find a file structure similar to what you saw in previous labs.  The top level directory contains `CONTRIBUTING.md`, `LICENSE.md`, `README.md`, and the directory with the code for this lab, `javacs-lab11`.

In the subdirectory `javacs-lab11/src/com/flatironschool/javacs` you'll find the source files for this lab:

    *  `WikiCrawler.java`, which contains starter code for your crawler.

    *  `WikiCrawlerTest.java`, which contains test code for `WikiCrawler`.

    *  `JedisIndex.java`, which is our solution to the previous lab.

You'll also find some of the helper classes we've used in previous lavs

    *  `JedisMaker.java`
    *  `WikiFetcher.java`
    *  `TermCounter.java`
    *  `WikiNodeIterable.java`

And as usual, in `javacs-lab11`, you'll find the Ant build file `build.xml`.

Before you run `JedisMaker`, you have to provide a file with information about your Redis server.  If you did this in the previous lab, you can just copy it over.  Otherwise you can find instructions in the previous lab.

Run `ant build` to compile the source files, then run `ant JedisMaker` to make sure it is configured to connect to your Redis server.

Now run `ant test` to run `WikiCrawlerTest`.  It should fail, because you have work to do!

Here's the beginning of the `WikiCrawler` class we provided:

```java
public class WikiCrawler {

	public final String source;
	private JedisIndex index;
	private Queue<String> queue = new LinkedList<String>();
	final static WikiFetcher wf = new WikiFetcher();

	public WikiCrawler(String source, JedisIndex index) {
		this.source = source;
		this.index = index;
		queue.offer(source);
	}

	public int queueSize() {
		return queue.size();
	}
```

The instance variables are

* `source` is the URL where we start crawling.

* `index` is the `JedisIndex` where the results should go.

* `queue` is a `LinkedList` where we keep track of URLs that have been discovered but not yet indexed.

* `wf` is the `WikiFetcher` we'll use to read and parse Web pages.

Your job is to fill in `crawl`.  Here's the prototype.

```java
	public String crawl(boolean testing) throws IOException {}
```

The parameter `testing` will be `true` when this method is called from `WikiCrawlerTest` and should be `false` otherwise.

When testing is `true`, the `crawl` method should:

*   Choose and remove a URL from the queue in FIFO order.

*   Read the contents of the page using `WikiFetcher.readWikipedia`, which reads cached copies of pages we have included in this repository for testing purposes (to avoid problems if the Wikipedia version changes).

*   It should index pages regardless of whether they are already indexed.

*   It should find all the internal links on the page and add them to the queue in the order they appear.   "Internal links" are links to other Wikipedia pages.

*   And it should return the URL of the page it indexed.

When testing is `false`, this method should:

*   Choose and remove a URL from the queue in FIFO order.

*   If the URL is already indexed, it should not index it again, and should return `null`.

*   Otherwise it should read the contents of the page using `WikiFetcher.fetchWikipedia`, which reads current content from the Web.

*   Then it should index the page, add links to the queue, and return the URL of the page it indexed.

`WikiCrawlerTest` loads the queue with about 200 links and then invokes `crawl` three times.  After each invocation, it checks the return value and the new length of the queue.

When your crawler is working as specified, this test should pass.  Good luck!


## Resources

*  [Stop words](https://en.wikipedia.org/wiki/Stop_words)

*  [Graphs](https://en.wikipedia.org/wiki/Graph_(discrete_mathematics))

*  [Graph traversal](https://en.wikipedia.org/wiki/Graph_traversal)

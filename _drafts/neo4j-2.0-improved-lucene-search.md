---
layout: post
title: Lucene Search in Neo4j
catagory: programming
tags:
    - neo4j
    - lucene
    - java
---


Neo4j 2.0
---------

Neo4j's 2.0 release introduced the concept of labels. I won't go into the
details of labels here. If you're not already familiar with the features of
Neo4j 2.0 you can read about it
[here](http://blog.neo4j.org/2013/04/nodes-are-people-too.html). 

### Indexes
Before 2.0, indexes had to be maintained manually. This was for all intents and
purposes impossible in any reasonably sane application code. With 2.0 comes
*real* indexes, tied to labels. They can be created as such

{% highlight java %}
graphDb.schema().indexFor(DynamicLabel.label("Label")).on("my_property").create();
{% endhighlight %}

The problem comes with how indexes are handled by the default index
implementation.

#### Lucene Analyzer
The analyzer that the default implementation uses is a keyword analyzer. Which is fine, but it doesn't use a lowercase filter. So queries must be case sensitive.
The fix for this is relatively easy, creating a lower case keyword analyzer.

{% highlight java %}
    public static final Analyzer KEYWORD_ANALYZER = new Analyzer()
    {
        @Override
        public TokenStream tokenStream( String fieldName, Reader reader )
        {
            return new LowerCaseFilter( LUCENE_VERSION, new KeywordTokenizer(reader) );
        }

        @Override
        public String toString()
        {
            return "LOWER_CASE_KEYWORD_ANALYZER";
        }
    };
{% endhighlight %}


#### Lucene Field Creation

When Neo4j creates fields, it tells lucene not to analyze them. That means
   that even fixing the above issue, new fields aren't sent through the lucene
   anaylzer. Again the fix for this is trivial. Just change 

{% highlight java %}
Field result = new Field( fieldIdentifier, value, store, NOT_ANALYZED );
{% endhighlight %}

to this

{% highlight java %}
Field result = new Field( fieldIdentifier, value, store, ANALYZED );
{% endhighlight %}
   
#### Neo4j's query code
   
The biggest problem comes with Neo4j's implementation of querying. Ignoring the implementation of how neo4j stores the indexed fields, they key thing to note is that any query you submit
to neo4j simply gets turned into a [TermQuery](http://lucene.apache.org/core/3_6_2/api/all/org/apache/lucene/search/TermQuery.html). There is no functionality to submit any custom query
or anything other than a string that gets turned into a singular query. I changed this so that most string queries get sent through lucene's query parser:

{% highlight java %}
    @Override
    Query encodeQuery( Object value )
    {
        try{
            QueryParser q = new QueryParser(LuceneDataSource.LUCENE_VERSION, key(), LuceneDataSource.LOWER_CASE_KEYWORD_ANALYZER);
            q.setAllowLeadingWildcard(true);
            return q.parse(value.toString());
        }catch (org.apache.lucene.queryParser.ParseException e){
            return new TermQuery( new Term( key(), value.toString() ) );
        }
    }
{% endhighlight %}

This works really well because if you just pass it a normal string, the QueryParser will return a basic TermQuery. The only backwards compatibility issue is that you now have to escape any
lucene special characters that were previously searched for literally.  


You can see the complete changes I made to the neo4j version I run on my github fork [https://github.com/Falmarri/neo4j/compare/lucene-query2](https://github.com/Falmarri/neo4j/compare/lucene-query2)

---
layout: post
title: Lessons learnt from building a REST API
categories: articles
---

Late last year, I built a REST based API for [ShermansTravel](http://www.shermanstravel.com) using Rails and [ActsAsAPI](https://github.com/fabrik42/acts_as_api). The API allowed us to tag and/or geotag any object, and was backed with [MongoDB](http://mongodb.org) and a little MySQL for authentication. The API itself was - and still is - sound, but unfortunately it turned out to be very slow. This was amplified even further due to its use by another REST based API.

These last few weeks have allowed me to revisit the application, and to make changes to speed things up, often-times resulting in a 400% speed increase, not to mention increased concurrency. This post will briefly detail the lessons learnt building a high trafficked API, and what changes I have made.

<!-- more -->

Please note, that this post is very Ruby-centric, so only considers what is available to the Rubyist.

### 1. Don't use Rails

Rails is awesome... for most things. But when you don't need any views or assets, or when you are not building a traditional web application, it's overkill. And you'll soon start finding that your app is slow. You have to remember that Rails is a full stack framework, but your API is most likely not.

If you really have to use a Framework, check out Sinatra, which has a much smaller footprint. Or peel it back even further, and create a bare bones Rack app. Which is a lot easier than you might think.


### 2. Don't use ActiveRecord

I hate to say this, but ActiveRecord is slow! In fact, most ORM's are usually much, much slower than using raw SQL. However, if your API uses MySQL heavily, you may not want to write the SQL yourself, and enjoy the huge conveniance that AR gives you. But you may be surprised how you can build yourself a little code that emulates a little of the AR API.

For example, in my API rewrite, I can still do this:

{% codeblock lang:ruby %}
deal = Deal.new
deal.where("category = 'hotel'").order('created_at DESC')
deal.run_query
{% endcodeblock %}

And this is my `Deal` model:

{% codeblock lang:ruby %}
class Deal
  def where(sql)
    @where_array ||= []
    @where_array << sql
  end

  def select(sql)
    @select_array ||= []
    @select_array << sql
  end

  def joins(sql)
    @joins_array ||= []
    @joins_array << sql
  end

  def order(sql)
    @order_array ||= []
    @order_array << sql
  end

  # Run the SQL query.
  def run_query
    connection.query build_sql
  end

  private

    # Builds the full SQL query.
    #
    # Returns the SQL statement as a String.
    def build_sql
      _select = @select_array || ['*']
      sql = "SELECT #{_select.join(', ')} FROM `places` "
      sql << @joins_array.join(' ') unless @joins_array.blank?
      sql << " WHERE #{@where_array.join(' AND ')}" unless @where_array.blank?
      sql << " ORDER BY #{@order_array.join(' ')}" unless @order_array.blank?

      unset_sql_variables

      sql
    end

    def unset_sql_variables
      @where_array, @select_array, @joins_array, @order_array = nil
    end

    def connection
      # your database connection
    end
end
{% endcodeblock %}

Very basic, but does the job nicely. My `connection` method simply uses the [MySQL2](https://github.com/brianmario/mysql2) Ruby Gem, which has very little overhead.


### 3. Increase concurrency and use EventMachine

This one is a biggie, but an absolute must, and is the biggest performance boost I have received so far.

[EventMachine](http://rubyeventmachine.com/) is a fast, simple event-processing library in a similar vein to Node.JS. But it's all Ruby! It allows you to take advantage of threaded network programming, thus increasing concurrency, scalability and performance.

Unfortunately, it has also involved some considerable rewriting, to ensure that my code is thread-safe, and that I take advantage of EventMachine as much as I can. However, the benefits have far exceeded this small cost. In fact, I now have a much smaller footprint all over my code, which also helps.

Not to get too specific, but my rewritten API uses the excellent [Goliath](http://postrank-labs.github.com/goliath/) Ruby web server, along with the em-synchrony, em-http-request and em-mongo gems to help me make my API asynchronous. And fast!


### 4. Use JSON... everywhere!

There is a reason why JSON is now almost the data format of the web. It's lightweight, simple and fast. And Ruby support for it is top-notch. Just add the [MultiJson](https://github.com/intridea/multi_json) and [OJ](https://github.com/ohler55/oj) gems to your Gemfile, and you won't get much faster.

Unless you have a real need to use XML or another data format, all you need is JSON.


#### In Conclusion...

There are obviously smaller things to think about when building an API, but the above four were the biggest changes I made to the API, and so far they have made a huge improvement.

Keep it mean, keep it lean, keep it kean!
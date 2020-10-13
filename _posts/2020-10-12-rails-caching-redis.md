---
title: Rails Model Caching with Redis
tags:
- redis
- Ruby
- rails
- caching-redis
style: fill
color: success
description: Lower level caches are very flexible and can work anywhere in the application.
---

{% include elements/figure.html image="https://miro.medium.com/max/785/1*fFt134G-jErKiohrosJlmA.png" caption="Caching With Redis" %}

<br />

> Model level caching is something that’s often ignored, even by seasoned developers. Much of it’s due to the misconception that, when you cache the views, you don’t need to cache at the lower levels. While it’s true that much of a bottleneck in the Rails world lies in the View layer, that’s not always the case .

Lower level caches are very flexible and can work anywhere in the application. In this tutorial, I’ll demonstrate how to cache your models with Redis.

# <span style="color:red">In this post </span>

* [How Caching Works ?](https://mohammed-basaleh.netlify.app/articles/rails-caching-redis#how-caching-works)
* [Why Redis ?](https://mohammed-basaleh.netlify.app/articles/rails-caching-redis#why-redis)
* [Getting Started](https://mohammed-basaleh.netlify.app/articles/rails-caching-redis#getting-started)
    * [Step 1: Create New Project :](https://mohammed-basaleh.netlify.app/articles/rails-caching-redis#step-1-create-new-project)
    * [Step 2: Init Redis:](https://mohammed-basaleh.netlify.app/articles/rails-caching-redis#step-2-init-redis)
    * [Step 3: Managing Cache:](https://mohammed-basaleh.netlify.app/articles/rails-caching-redis#step-3-managing-cache)
* [Conclusion](https://mohammed-basaleh.netlify.app/articles/rails-caching-redis#conclusion)


#  <span style="color:red">How Caching Works?</span>
Traditionally, accessing disk has been expensive. Trying to access data from the disk frequently will have an adverse impact on performance. To counter this, we can implement a caching layer in between your application and the **database server**.

A caching layer doesn’t hold any data at first. When it receives a request for the data, it calls the database and stores the result in memory (`the cache`). All subsequent requests will be served from the cache layer, so the unnecessary roundtrip to the **database server** is avoided, improving performance.

#  <span style="color:red">Why Redis?</span>
Redis is an in-memory, key-value store. It’s blazingly fast and data retrieval is almost instantaneous. Redis supports advanced data structures like lists, hashes, sets, and can persist to disk.

> I find `Redis` very simple to setup and easy to administer. Also, if you are using resque or **Sidekiq** for managing your background jobs, you probably have Redis installed already.


#  <span style="color:red">Getting Started</span>
<br>
#### <span style="color:TEAL"> Step 1: Create New Project: </span>
Let's build new project or  if you already have an existing project, go to step 

> rails new demo_caching_redis -d postgresql


then build new **Employee** model that contains an imaginary number of employees

> rails g scaffold Employee name:string description:text

after create scaffold, Let's take a look at the model and controller

<pre tabindex="0" class=" language-ruby"><code class=" language-ruby"><span class="token comment"># app/models/employee.rb</span>

<span class="token keyword">class</span> <span class="token class-name">Employee</span> <span class="token operator">&lt;</span> <span class="token constant">ApplicationRecord</span>
<span class="token keyword">end</span>

</code></pre>

<pre tabindex="0" class=" language-ruby"><code class=" language-ruby"><span class="token comment"># app/controllers/employees_controller.rb</span>

<span class="token keyword">class</span> <span class="token class-name">EmployeeController</span> <span class="token operator">&lt;</span> <span class="token constant">ApplicationController</span>
  <span class="token keyword">def</span> <span class="token method-definition"><span class="token function">index</span></span>
    <span class="token variable">@employees</span> <span class="token operator">=</span> <span class="token constant">Employee</span><span class="token punctuation">.</span>all
  <span class="token keyword">end</span>
<span class="token keyword">end</span>
</code></pre>

now we need to run our db:migrate

> rails db:migrate

Now in your db/seed.rb, we’ll create some dummy data.
<pre tabindex="0" class=" language-ruby"><code class=" language-ruby"><span class="token comment"># app/db/seed.rb</span>
100_000.time do
 Employee.create(name: Faker::Name.name, descriprion: Faker::StarWars:qoute)
end
</code></pre>
Until this point we’ve created 100,000 dummy records.

And now we’ll change our router adding root `employees#index` and `resources employees`

<pre tabindex="0" class=" language-ruby"><code class=" language-ruby"><span class="token comment"># app/config/routes.rb</span>
Rails.application.routes.draw do
  root 'employees#index'
  resources :employees
  # For details on the DSL available within this file, see https://guides.rubyonrails.org/routing.html
end
</code></pre>


Now we need to run our server, the `rails server` command launches a web server named Puma which comes bundled with Rails.

> rails server 

Now type `localhost:3000` in your browser, As we can see, our project took a long time.
> Completed 200 OK in 12261ms

Since the metadata models are unlikely to change that often it makes sense to avoid unnecessary database roundtrips. This is where lower level caching comes in.
<br>

#### <span style="color:TEAL"> Step 2: Init Redis: </span>
* There is a Ruby client for Redis that helps us to connect to the redis instance easily:

<pre tabindex="0" class=" language-ruby"><code class=" language-ruby"><span class="token comment"># Gemfile</span>
gem 'redis'
gem 'redis-namespace'
gem 'redis-rails'
gem 'redis-rack-cache'
</code></pre>

* Once these gems are installed, instruct Rails to use Redis as the cache store:

<pre tabindex="0" class=" language-ruby"><code class=" language-ruby"><span class="token comment"># config/application.rb</span>
#...........
config.cache_store = :redis_store, 'redis://localhost:6379/0/cache', { expires_in: 120.minutes }
#.........
</code></pre>

* The redis-namespace gem allows us to create nice a wrapper around Redis:
<pre tabindex="0" class=" language-ruby"><code class=" language-ruby"><span class="token comment"># config/initializers/redis.rb </span>
$redis = Redis::Namespace.new('your_name_site', redis: Redis.new)
</code></pre>

> All the Redis functionality is now available across the entire app through the `$redis` global. Here’s an example of how to access the values in the redis server (fire up a Rails console):

<pre tabindex="0" class=" language-ruby"><code class=" language-ruby">$redis.set(:test_key, 'Hello World!)</code></pre>

>This command will create a new key called `:test_key` in Redis with the value `Hello World`. To fetch this value, just do:

<pre tabindex="0" class=" language-ruby"><code class=" language-ruby">$redis.get(:test_key)</code></pre>
<br>
### Now that we have the basics, let’s start writing our coding
<br>
<pre tabindex="0" class=" language-ruby"><code class=" language-ruby"><span class="token comment"># app/controllers/employees_controller.rb</span>
<span class="token keyword">class</span> <span class="token class-name">EmployeeController</span> <span class="token operator">&lt;</span> <span class="token constant">ApplicationController</span>
  <span class="token keyword">def</span> <span class="token method-definition"><span class="token function">index</span></span>
    employees <span class="token operator">=</span>  <span class="token variable">$redis</span><span class="token punctuation">.</span>get<span class="token punctuation">(</span><span class="token string">:employees</span><span class="token punctuation">)</span>
    <span class="token keyword">if</span> employees<span class="token punctuation">.</span><span class="token keyword">nil</span><span class="token operator">?
      </span>employees <span class="token operator">=</span> <span class="token constant">Category</span><span class="token punctuation">.</span>all<span class="token punctuation">.</span>to_json
      <span class="token variable">$redis</span><span class="token punctuation">.</span>set<span class="token punctuation">(</span><span class="token string">:employees</span><span class="token punctuation">,</span> employees<span class="token punctuation">)</span>
    <span class="token keyword">end</span>
    <span class="token variable">@employees</span> <span class="token operator">=</span> <span class="token constant">JSON</span><span class="token punctuation">.</span>parse employees
  <span class="token keyword">end</span>
<span class="token keyword">end</span>
</code></pre>
<br>

> The first time this code executes there won’t be anything in memory/cache. So, we ask Rails to fetch it from the database and then push it to redis. **Notice** the `to_json` call? When writing objects to Redis, The simplest way is to save them as a `JSON` encoded string. To decode, simply use `JSON.parse` .
> <span style="color:red"> However, this comes in with an unintended side effect. When we’re retrieving the values, a simple object notation won’t work. We need to update the views to use the hash syntax to display the employees :</span>

<br>

<pre tabindex="0" class=" language-ruby"><code class=" language-ruby"><span class="token comment"># app/views/employees/index.html.haml</span>
.
.
<% @employees.each do |employee| %>
 <td><%= employee["name"] %></td>
 <td><%= employee["description"] %></td>
<% end %>
</code></pre>

Fire up the browser again and see if there is a performance difference. The first time, we still hit the database, but on subsequent reloads, the database is not used at all. All future requests will be loaded from the cache. That’s a lot of savings for a simple change  .

Now type `localhost:3000` in your browser, As we can see, our project take a look at the time taken, you'll notice the big difference .
> Completed 200 OK in 283ms

<br>
#### <span style="color:TEAL"> Step 3: Managing Cache: </span>

When employee data is updated, this change is not reflected in our views. This is because we’ve bypassed accessing the database and all values are served from the cache. Alas, the cache is now stale and the updated data won’t be available until Redis is restarted.

If you find there are two ways to overcome this and they are:
* Expiring the cache periodically:
<pre tabindex="0" class=" language-ruby"><code class=" language-ruby"><span class="token comment"># app/controllers/employees_controller.rb</span>
<span class="token keyword">class</span> <span class="token class-name">EmployeeController</span> <span class="token operator">&lt;</span> <span class="token constant">ApplicationController</span>
  <span class="token keyword">def</span> <span class="token method-definition"><span class="token function">index</span></span>
    employees <span class="token operator">=</span>  <span class="token variable">$redis</span><span class="token punctuation">.</span>get<span class="token punctuation">(</span><span class="token string">:employees</span><span class="token punctuation">)</span>
    <span class="token keyword">if</span> employees<span class="token punctuation">.</span><span class="token keyword">nil</span><span class="token operator">?
      </span>employees <span class="token operator">=</span> <span class="token constant">Employee</span><span class="token punctuation">.</span>all<span class="token punctuation">.</span>to_json
      <span class="token variable">$redis</span><span class="token punctuation">.</span>set<span class="token punctuation">(</span><span class="token string">:employees</span><span class="token punctuation">,</span> employees<span class="token punctuation">)</span>
      <span class="token comment"># Expire the cache, every 3 hours</span>
      <span class="token variable">$redis</span><span class="token punctuation">.</span>expire<span class="token punctuation">(</span><span class="token string">:employees</span><span class="token punctuation">,</span><span class="token number">3.</span>hour<span class="token punctuation">.</span>to_i<span class="token punctuation">)</span>
    <span class="token keyword">end</span>
    <span class="token variable">@employees</span> <span class="token operator">=</span> <span class="token constant">JSON</span><span class="token punctuation">.</span>parse employees
  <span class="token keyword">end</span>
<span class="token keyword">end</span>
</code></pre>
This will expire the cache every `3 hours`. While this works for most scenarios, the data in the cache will now lag the database. 
* Through callback (after save)
<pre tabindex="0" class=" language-ruby"><code class=" language-ruby"><span class="token comment"># app/models/employee.rb</span>

<span class="token keyword">class</span> <span class="token class-name">Employee</span> <span class="token operator">&lt;</span> <span class="token constant">ApplicationRecord</span>
#...........
  after_save :clear_cache

  def clear_cache
    $redis.del :employees
  end
#...........
<span class="token keyword">end</span>
</code></pre>
Every time the model is updated, we’re instructing Rails to clear the cache. This will ensure that the cache is always up to date.

<br>


# <span style="color:red">Conclusion :</span>
Lower level caching is very simple and, when properly used, it is very rewarding. It can instantaneously boost your system’s performance with minimal effort. All the [demo caching redis](https://github.com/MohamedBasaleh/demo-caching-redis) in this article are available on Github.

Hope enjoyed reading this. Please share your thoughts in the comments.

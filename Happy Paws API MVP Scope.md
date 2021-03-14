---
title: Happy Paws API MVP Scope
author: M. Tucker Bradford
extensions:
  preset: gfm

---

<h1 id="happy-paws-api-mvp-scope">Happy Paws API MVP Scope</h1>
<h2 id="overview">Overview</h2>
<p>Happy Paws has committed to providing an API for managing internal promotional campaigns. This minimum viable product (MVP) will detail the core needs of that product, and is designed to adapt and scale with the customer’s growth. Certain design decisions have been made to minimize the initial investment in the product, while avoiding potential future scalability issues, and other design snares.</p>
<p>We will use the Ruby on Rails framework, as it is the fastest and most flexible prototyping framework available today. Wherever practical we will leverage in house libraries, but in certain areas (such as authentication and authorization) we will leverage gems (external libraries) to speed up development and enhance performance and security.</p>
<p>Initially the product will consist of 6 models. These models are detailed in Appendix A. The core models are the Audience (User), Tier (Group), and Campaign. Roles are used for Authorization, and a join table is used to connect the many to many relationships of Audience and Campaign.</p>
<h2 id="data-model">Data Model</h2>
<p>As reflected in the requirements document, an AudienceMember can belong to many Campaigns, via a Tier. A Tier, in turn, can belong to another Tier in a parent/child relationship. This parent/child relationship allows future development of nested features, and presents a visible and flexible hierarchy for Tiers, that is reflected in the data.</p>
<p>There are initially two Roles. Administrator, and [pet] Owner. An AudienceMember may only have one Role at this time.</p>
<p>All persisted data will be stored in a PostgreSQL backend.</p>
<h2 id="frontend">Frontend</h2>
<p>The initial frontend will be provided through the Rails framework leveraging server side rendering for faster prototyping. Engineers will take all possible care to abstract logic away from the controller and into service objects, so that logic may be shared with either RESTful or GraphQL API controllers in the future.</p>
<h3 id="administrator-experience">Administrator Experience</h3>
<p>For the MVP, the Administrator functionality will be handled through the Administrate gem. The Administrate gem will enable the creation, deletion, and modification of AudienceMembers, Tiers, and Campaigns, and will automatically handle the joining of those relationships. While this is a less visually elegant solution than a custom made Admin UI, it will be possible to produce it quickly, and the time saved can be used to flesh out the API, which is a crucial part of our growth plan. Reports can be created through Administrate as well, and exported as JSON or CSV.</p>
<h3 id="reporting">Reporting</h3>
<p>Two simple reports will be created through a <code>ReportsController</code></p>
<p>The  controller will present a table (or csv file) containing the total count of all campaigns executed grouped by month. A presenter will key off of the <code>type</code> param, to reduce the data into <code>count</code> and <code>cost</code> reports. Reports will be displayed as a table (in the case of an html request) or csv file (in the case of a csv request)</p>
<pre class=" language-ruby"><code class="prism  language-ruby"><span class="token keyword">def</span> show
  <span class="token variable">@campaigns</span> <span class="token operator">=</span> <span class="token constant">Campaign</span><span class="token punctuation">.</span><span class="token function">joins</span><span class="token punctuation">(</span>tiers<span class="token punctuation">:</span> <span class="token symbol">:audience_members</span><span class="token punctuation">)</span><span class="token punctuation">.</span><span class="token function">group_by</span><span class="token punctuation">(</span><span class="token operator">&amp;</span><span class="token symbol">:mailing_date</span><span class="token punctuation">)</span>
  respond_to <span class="token keyword">do</span> <span class="token operator">|</span>format<span class="token operator">|</span>
    format<span class="token punctuation">.</span>csv  <span class="token punctuation">{</span> render csv<span class="token punctuation">:</span> <span class="token variable">@campaigns</span> <span class="token punctuation">}</span>
    format<span class="token punctuation">.</span>html
  <span class="token keyword">end</span>
<span class="token keyword">end</span>
</code></pre>
<h2 id="api">API</h2>
<p>Though the MVP for Happy Paws will have a server side rendered frontend, our goal is to move towards a React (client side) frontend in the near future. In preparation for that, we will build our API on the GraphQL model initially (though as mentioned earlier, we will abstract concerns such that a RESTful API would be possible in the future).</p>
<h3 id="authentication">Authentication</h3>
<p>The graphql implementation will be facilitated by the graphql-rails gem. This will allow rapid and flexible development, and leaves room for performance improvements in the future. API authentication will be provided by this home rolled JWT implementation:</p>
<pre class=" language-ruby"><code class="prism  language-ruby"><span class="token keyword">class</span> <span class="token class-name">JsonWebToken</span>
  <span class="token keyword">class</span> <span class="token operator">&lt;</span><span class="token operator">&lt;</span> <span class="token keyword">self</span>
    <span class="token keyword">def</span> <span class="token function">encode</span><span class="token punctuation">(</span>payload<span class="token punctuation">,</span> exp <span class="token operator">=</span> <span class="token number">24</span><span class="token punctuation">.</span>hours<span class="token punctuation">.</span>from_now<span class="token punctuation">)</span>
      payload<span class="token punctuation">[</span><span class="token symbol">:exp</span><span class="token punctuation">]</span> <span class="token operator">=</span> exp<span class="token punctuation">.</span>to_i
      <span class="token constant">JWT</span><span class="token punctuation">.</span><span class="token function">encode</span><span class="token punctuation">(</span>payload<span class="token punctuation">,</span> <span class="token constant">ENV</span><span class="token punctuation">[</span><span class="token string">'JWT_SECRET_KEY'</span><span class="token punctuation">]</span><span class="token punctuation">)</span>
    <span class="token keyword">end</span>

    <span class="token keyword">def</span> <span class="token function">decode</span><span class="token punctuation">(</span>token<span class="token punctuation">)</span>
      body <span class="token operator">=</span> <span class="token constant">JWT</span><span class="token punctuation">.</span><span class="token function">decode</span><span class="token punctuation">(</span>token<span class="token punctuation">,</span> <span class="token constant">ENV</span><span class="token punctuation">[</span><span class="token string">'JWT_SECRET_KEY'</span><span class="token punctuation">]</span><span class="token punctuation">)</span><span class="token punctuation">[</span><span class="token number">0</span><span class="token punctuation">]</span>
      <span class="token constant">HashWithIndifferentAccess</span><span class="token punctuation">.</span><span class="token keyword">new</span> <span class="token class-name">body</span>
    <span class="token keyword">rescue</span> <span class="token constant">StandardError</span>
      <span class="token keyword">nil</span>
    <span class="token keyword">end</span>
  <span class="token keyword">end</span>
<span class="token keyword">end</span>
</code></pre>
<p>This code will be called by service objects (namespaced as <code>commands</code> in this context) which will in turn be called by the authentication controller. This separation of concern will allow us to reuse the JWT functionality in other contexts later.</p>
<h3 id="graphql-types">GraphQL Types</h3>
<p>Initially our data will be reflected in 3 Types, which can be graphed (through GraphQL queries) into complex data structures which reflect our frontend's needs. These types will be Audience Members, Tiers, &amp; Campaigns.</p>
<pre class=" language-ruby"><code class="prism  language-ruby"><span class="token comment"># app/graphql/types/audience_type.rb</span>

<span class="token keyword">module</span> <span class="token constant">Types</span>
  <span class="token keyword">class</span> <span class="token class-name">AudienceMemberType</span> <span class="token operator">&lt;</span> <span class="token constant">Types</span><span class="token punctuation">:</span><span class="token symbol">:BaseObject</span>
    field <span class="token symbol">:id</span><span class="token punctuation">,</span> <span class="token constant">ID</span><span class="token punctuation">,</span> null<span class="token punctuation">:</span> <span class="token keyword">false</span>
    field <span class="token symbol">:full_name</span><span class="token punctuation">,</span> <span class="token builtin">String</span><span class="token punctuation">,</span> null<span class="token punctuation">:</span> <span class="token keyword">false</span>
    field <span class="token symbol">:created_at</span><span class="token punctuation">,</span> <span class="token constant">ISO8601DateTime</span><span class="token punctuation">,</span> null<span class="token punctuation">:</span> <span class="token keyword">false</span>
    field <span class="token symbol">:tier</span><span class="token punctuation">,</span> <span class="token constant">Types</span><span class="token symbol">:TierType</span><span class="token punctuation">,</span> null<span class="token punctuation">:</span> <span class="token keyword">true</span>
    field <span class="token symbol">:campaigns</span><span class="token punctuation">,</span> <span class="token punctuation">[</span><span class="token constant">Types</span><span class="token symbol">:CampaignType</span><span class="token punctuation">]</span><span class="token punctuation">,</span> null<span class="token punctuation">:</span> <span class="token keyword">true</span>
    field <span class="token symbol">:role</span><span class="token punctuation">,</span> <span class="token constant">Types</span><span class="token symbol">:RoleType</span><span class="token punctuation">,</span> null<span class="token punctuation">:</span> <span class="token keyword">false</span>
  <span class="token keyword">end</span>

<span class="token keyword">end</span>

<span class="token comment"># app/graphql/types/tier_type.rb</span>
modules <span class="token constant">Types</span>
  <span class="token keyword">class</span> <span class="token class-name">TierType</span> <span class="token operator">&lt;</span> <span class="token constant">Types</span><span class="token punctuation">:</span><span class="token symbol">:BaseObject</span>
    field <span class="token symbol">:id</span><span class="token punctuation">,</span> <span class="token constant">ID</span><span class="token punctuation">,</span> null<span class="token punctuation">:</span> <span class="token keyword">false</span>
    field <span class="token symbol">:name</span><span class="token punctuation">,</span> <span class="token builtin">String</span><span class="token punctuation">,</span> null<span class="token punctuation">:</span> <span class="token keyword">false</span>
    field <span class="token symbol">:details</span><span class="token punctuation">,</span> <span class="token builtin">String</span><span class="token punctuation">,</span> null<span class="token punctuation">:</span> <span class="token keyword">false</span>
    field <span class="token symbol">:parent</span><span class="token punctuation">,</span> <span class="token constant">Types</span><span class="token symbol">:TierType</span><span class="token punctuation">,</span> null<span class="token punctuation">:</span> <span class="token keyword">true</span>
    field <span class="token symbol">:campaigns</span><span class="token punctuation">,</span> <span class="token punctuation">[</span><span class="token constant">Types</span><span class="token symbol">:CampaignType</span><span class="token punctuation">]</span><span class="token punctuation">,</span> null<span class="token punctuation">:</span> <span class="token keyword">true</span>
  <span class="token keyword">end</span>
<span class="token keyword">end</span>
  
<span class="token comment"># …etc</span>

</code></pre>
<p>An example curl request that would return all Campaigns, their Tiers, and AudienceMembers</p>
<pre class=" language-graphql"><code class="prism  language-graphql"><span class="token comment">## Request</span>
curl -X <span class="token string">"POST"</span> <span class="token string">"https://happypaws.com/api/v1/"</span> \
     -H '<span class="token attr-name">Authorization</span><span class="token punctuation">:</span> Bearer asdcm329cna823pasdcg23' \
     -H 'Content-<span class="token attr-name">Type</span><span class="token punctuation">:</span> application/json; charset<span class="token operator">=</span>utf-<span class="token number">8</span>' \
     -d $'<span class="token punctuation">{</span>
	<span class="token string">"query"</span><span class="token punctuation">:</span> <span class="token string">"query{campaigns{name recipient_count promo_code mailing_date paper_format{name cost}tiers{id parent{name details}audience_members{nodes{full_name mailing_address}}}}}"</span><span class="token punctuation">,</span>
	<span class="token string">"variables"</span><span class="token punctuation">:</span> <span class="token punctuation">{</span><span class="token punctuation">}</span>
<span class="token punctuation">}</span>'

</code></pre>
<p>Or the corresponding GraphQL query</p>
<pre class=" language-graphql"><code class="prism  language-graphql"><span class="token keyword">query</span> <span class="token punctuation">{</span>
  campaigns <span class="token punctuation">{</span>
    name
    recipient_count
    promo_code
    mailing_date
    
    paper_format <span class="token punctuation">{</span>
      name
      cost
    <span class="token punctuation">}</span>
    tiers <span class="token punctuation">{</span>
      id
      parent <span class="token punctuation">{</span>
        name
        details
      <span class="token punctuation">}</span>
      audience_members <span class="token punctuation">{</span>
        nodes <span class="token punctuation">{</span>
          full_name
          mailing_address
        <span class="token punctuation">}</span>
      <span class="token punctuation">}</span>
    <span class="token punctuation">}</span>
  <span class="token punctuation">}</span>
<span class="token punctuation">}</span>

</code></pre>
<hr>
<h2 id="out-of-scope">Out of Scope</h2>
<p>There are a number of concerns that will become more pressing as the user base and application functionality scale. Some examples are (but are not limited to):</p>
<ul>
<li><strong>Caching</strong>: Rails has built in database caching, but GraphQL caching, and other<br>
types of object caching will not be implemented in this MVP.</li>
<li><strong>Backgrounding</strong>: Eventually we will utilize Redis/Sidekiq for backgrounding,<br>
but at this point the anticipated load times are nominal and well within<br>
tolerances for responsiveness.</li>
<li><strong>Pets</strong>: It is anticipated the Happy Paws will desire the ability to add a<br>
Pet (or pets) to the AudienceMember. If this is desirable, we must consider<br>
moving the Tier relationship to the Pet model. This is out of scope for this<br>
MVP</li>
<li><strong>Frontend</strong>: The requirements document makes no mention of a customer facing<br>
frontend, so one will not be provided. If the customer requires a placeholder<br>
(Welcome Page) or other customer facing service, that will have to be scoped<br>
separately.<br>
<br><br>
<br><br>
<br><br>
<br><br>
<br><br>
<br></li>
</ul>
<hr>
<h2 id="appendix-a">Appendix A</h2>
<p><img src="https://user-images.githubusercontent.com/63799/111077483-5424fa80-84c7-11eb-9eff-4253a3025640.png" alt="happy paws uml diagram"></p>


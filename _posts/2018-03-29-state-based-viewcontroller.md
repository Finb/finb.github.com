---
layout: post
categories: 
title: 单向数据流动的函数式 View Controller
date: 2018-03-29 14:35:39 +0800
description: 
keywords: 
catalog: true
multilingual: false
tags: 
---

<span style='margin-bottom:10px;'>
转载自： <a href="https://onevcat.com/2017/07/state-based-viewcontroller/">https://onevcat.com/2017/07/state-based-viewcontroller/</a> 
</span>
  <section class="post">
    <p>View Controller 向来是 MVC (Model-View-View Controller) 中最让人头疼的一环，MVC 架构本身并不复杂，但开发者很容易将大量代码扔到用于协调 View 和 Model 的 Controller 中。你不能说这是一种错误，因为 View Controller 所承担的本来就是胶水代码和业务逻辑的部分。但是，持续这样做必定将导致 Model View Controller 变成 Massive View Controller，代码也就一天天烂下去，直到没人敢碰。</p>

<p>对于采用 MVC 架构的项目来说，其实最大的挑战在于维护 Controller。而想要有良好维护的 Controller，最大的挑战又在于保持良好的测试覆盖。因为往往 View Controller 中会包含很多状态，而且会有不少异步操作和用户触发的事件，所以测试 Controller 从来都不是一件简单的事情。</p>

<blockquote>
  <p>这一点对于一些类似的其他架构也是一样的。比如 MVVM 或者 VIPER，广义上它们其实都是 MVC，只不过使用 View Model 或者 Presenter 来做 Controller 而已。它们对应的控制器的职责依然是协调 Model 和 View。</p>
</blockquote>

<p>在这篇文章里，我会先实现一个很常见的 MVC 架构，然后对状态和状态改变的部分进行抽象及重构，最终得到一个纯函数式的易于测试的 View Controller 类。希望通过这个例子，能够给你在日常维护 View Controller 的工作中带来一些启示或者帮助。</p>

<p>如果你对 React 和 Redux 有所了解的话，文中的一些方法可能你会很熟悉。不过即使你不了解它们，也并不会妨碍你理解本文。我不会去细究概念上的东西，而会从一个大家都所熟知的例子开始进行介绍，所以完全不用担心。你可能需要对 Swift 有一些了解，本文会涉及一些基本的值类型和引用类型的区别，如果你对此不是很明白的话，可以参看一些其他资料，比如我以前写的<a href="http://swifter.tips/value-reference/">这篇文章</a>。</p>

<p>整个示例项目我放在了 <a href="https://github.com/onevcat/ToDoDemo">GitHub</a> 上，你可以在各个分支中找到对应的项目源码。</p>

<h2 id="传统-mvc-实现">传统 MVC 实现</h2>

<p>我们用一个经典的 ToDo 应用作为示例。这个项目可以从网络加载待办事项，我们通过输入文本进行添加，或者点击对应条目进行删除：</p>

<center>
<video width="272" height="480" controls="">
  <source src="/assets/images/2017/todo-video.mp4" type="video/mp4">
</video>
</center>

<p>注意几个细节：</p>

<ol>
  <li>打开应用后加载已有待办列表时花费了一些时间，一般来说，我们会从网络请求进行加载，这应该是一个异步操作。在示例项目里，我们不会真的去进行网络请求，而是使用一个本地存储来模拟这个过程。</li>
  <li>标题栏的数字表示当前已有的待办项目，随着待办的增减，这个数字会相应变化。</li>
  <li>可以使用第一个 cell 输入，并用右上角的加号添加一个待办。我们希望待办事项的标题长度至少为三个字符，在不满足长度的时候，添加按钮不可用。</li>
</ol>

<p>实现这些并没有太大难度，一个刚入门 iOS 的新人也应该能毫无压力搞定。我们先来实现模拟异步获取已有待办的部分。新建一个文件 ToDoStore.swift:</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">import</span> <span class="kt">Foundation</span>

<span class="k">let</span> <span class="nv">dummy</span> <span class="o">=</span> <span class="p">[</span>
    <span class="s">"Buy the milk"</span><span class="p">,</span>
    <span class="s">"Take my dog"</span><span class="p">,</span>
    <span class="s">"Rent a car"</span>
<span class="p">]</span>

<span class="kd">struct</span> <span class="kt">ToDoStore</span> <span class="p">{</span>
    <span class="kd">static</span> <span class="k">let</span> <span class="nv">shared</span> <span class="o">=</span> <span class="kt">ToDoStore</span><span class="p">()</span>
    <span class="kd">func</span> <span class="nf">getToDoItems</span><span class="p">(</span><span class="nv">completionHandler</span><span class="p">:</span> <span class="p">(([</span><span class="kt">String</span><span class="p">])</span> <span class="o">-&gt;</span> <span class="kt">Void</span><span class="p">)?)</span> <span class="p">{</span>
        <span class="kt">DispatchQueue</span><span class="o">.</span><span class="n">main</span><span class="o">.</span><span class="nf">asyncAfter</span><span class="p">(</span><span class="nv">deadline</span><span class="p">:</span> <span class="o">.</span><span class="nf">now</span><span class="p">()</span> <span class="o">+</span> <span class="mi">2</span><span class="p">)</span> <span class="p">{</span>
            <span class="nf">completionHandler</span><span class="p">?(</span><span class="n">dummy</span><span class="p">)</span>
        <span class="p">}</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

<p>为了简明，我们使用简单的 <code class="highlighter-rouge">String</code> 来代表一条待办。这里我们等待了两秒后才调用回调，传回一组预先定义的待办事项。</p>

<p>由于整个界面就是一个 Table View，所以我们创建一个 <code class="highlighter-rouge">UITableViewController</code> 子类来实现需求。在 TableViewController.swift 中，我们定义一个属性 <code class="highlighter-rouge">todos</code> 来存放需要显示在列表中的待办事项，然后在 <code class="highlighter-rouge">viewDidLoad</code> 里从 <code class="highlighter-rouge">ToDoStore</code> 中进行加载并刷新 <code class="highlighter-rouge">tableView</code>：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">class</span> <span class="kt">TableViewController</span><span class="p">:</span> <span class="kt">UITableViewController</span> <span class="p">{</span>

    <span class="k">var</span> <span class="nv">todos</span><span class="p">:</span> <span class="p">[</span><span class="kt">String</span><span class="p">]</span> <span class="o">=</span> <span class="p">[]</span>
    
    <span class="k">override</span> <span class="kd">func</span> <span class="nf">viewDidLoad</span><span class="p">()</span> <span class="p">{</span>
        <span class="k">super</span><span class="o">.</span><span class="nf">viewDidLoad</span><span class="p">()</span>

        <span class="kt">ToDoStore</span><span class="o">.</span><span class="n">shared</span><span class="o">.</span><span class="n">getToDoItems</span> <span class="p">{</span> <span class="p">(</span><span class="n">data</span><span class="p">)</span> <span class="k">in</span>
            <span class="k">self</span><span class="o">.</span><span class="n">todos</span> <span class="o">+=</span> <span class="n">data</span>
            <span class="k">self</span><span class="o">.</span><span class="n">title</span> <span class="o">=</span> <span class="s">"TODO - (</span><span class="se">\(</span><span class="k">self</span><span class="o">.</span><span class="n">todos</span><span class="o">.</span><span class="n">count</span><span class="se">)</span><span class="s">)"</span>
            <span class="k">self</span><span class="o">.</span><span class="n">tableView</span><span class="o">.</span><span class="nf">reloadData</span><span class="p">()</span>
        <span class="p">}</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

<p>当然，我们现在需要提供 <code class="highlighter-rouge">UITableViewDataSource</code> 的相关方法。首先，我们的 Table View 有两个 section，一个负责输入新的待办，另一个负责展示现有的条目。为了让代码清晰表意自解释，我选择在 <code class="highlighter-rouge">TableViewController</code> 里内嵌一个 <code class="highlighter-rouge">Section</code> 枚举：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">class</span> <span class="kt">TableViewController</span><span class="p">:</span> <span class="kt">UITableViewController</span> <span class="p">{</span>
    <span class="kd">enum</span> <span class="kt">Section</span><span class="p">:</span> <span class="kt">Int</span> <span class="p">{</span>
        <span class="k">case</span> <span class="n">input</span> <span class="o">=</span> <span class="mi">0</span><span class="p">,</span> <span class="n">todos</span><span class="p">,</span> <span class="n">max</span>
    <span class="p">}</span>
    
    <span class="c1">//...</span>
<span class="p">}</span>
</code></pre></div></div>

<p>这样，我们就可以实现 <code class="highlighter-rouge">UITableViewDataSource</code> 所需要的方法了：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">class</span> <span class="kt">TableViewController</span><span class="p">:</span> <span class="kt">UITableViewController</span> <span class="p">{</span>
    <span class="k">override</span> <span class="kd">func</span> <span class="nf">numberOfSections</span><span class="p">(</span><span class="k">in</span> <span class="nv">tableView</span><span class="p">:</span> <span class="kt">UITableView</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="kt">Int</span> <span class="p">{</span>
        <span class="k">return</span> <span class="kt">Section</span><span class="o">.</span><span class="n">max</span><span class="o">.</span><span class="n">rawValue</span>
    <span class="p">}</span>
    
    <span class="k">override</span> <span class="kd">func</span> <span class="nf">tableView</span><span class="p">(</span><span class="n">_</span> <span class="nv">tableView</span><span class="p">:</span> <span class="kt">UITableView</span><span class="p">,</span> <span class="n">numberOfRowsInSection</span> <span class="nv">section</span><span class="p">:</span> <span class="kt">Int</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="kt">Int</span> <span class="p">{</span>
        <span class="k">guard</span> <span class="k">let</span> <span class="nv">section</span> <span class="o">=</span> <span class="kt">Section</span><span class="p">(</span><span class="nv">rawValue</span><span class="p">:</span> <span class="n">section</span><span class="p">)</span> <span class="k">else</span> <span class="p">{</span>
            <span class="nf">fatalError</span><span class="p">()</span>
        <span class="p">}</span>
        <span class="k">switch</span> <span class="n">section</span> <span class="p">{</span>
        <span class="k">case</span> <span class="o">.</span><span class="nv">input</span><span class="p">:</span> <span class="k">return</span> <span class="mi">1</span>
        <span class="k">case</span> <span class="o">.</span><span class="nv">todos</span><span class="p">:</span> <span class="k">return</span> <span class="n">todos</span><span class="o">.</span><span class="n">count</span>
        <span class="k">case</span> <span class="o">.</span><span class="nv">max</span><span class="p">:</span> <span class="nf">fatalError</span><span class="p">()</span>
        <span class="p">}</span>
    <span class="p">}</span>
    
    <span class="k">override</span> <span class="kd">func</span> <span class="nf">tableView</span><span class="p">(</span><span class="n">_</span> <span class="nv">tableView</span><span class="p">:</span> <span class="kt">UITableView</span><span class="p">,</span> <span class="n">cellForRowAt</span> <span class="nv">indexPath</span><span class="p">:</span> <span class="kt">IndexPath</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="kt">UITableViewCell</span> <span class="p">{</span>
        <span class="k">guard</span> <span class="k">let</span> <span class="nv">section</span> <span class="o">=</span> <span class="kt">Section</span><span class="p">(</span><span class="nv">rawValue</span><span class="p">:</span> <span class="n">indexPath</span><span class="o">.</span><span class="n">section</span><span class="p">)</span> <span class="k">else</span> <span class="p">{</span>
            <span class="nf">fatalError</span><span class="p">()</span>
        <span class="p">}</span>
        
        <span class="k">switch</span> <span class="n">section</span> <span class="p">{</span>
        <span class="k">case</span> <span class="o">.</span><span class="nv">input</span><span class="p">:</span>
            <span class="c1">// 返回 input cell</span>
        <span class="k">case</span> <span class="o">.</span><span class="nv">todos</span><span class="p">:</span>
            <span class="c1">// 返回 todo item cell</span>
            <span class="k">let</span> <span class="nv">cell</span> <span class="o">=</span> <span class="n">tableView</span><span class="o">.</span><span class="nf">dequeueReusableCell</span><span class="p">(</span><span class="nv">withIdentifier</span><span class="p">:</span> <span class="n">todoCellReuseId</span><span class="p">,</span> <span class="nv">for</span><span class="p">:</span> <span class="n">indexPath</span><span class="p">)</span>
            <span class="n">cell</span><span class="o">.</span><span class="n">textLabel</span><span class="p">?</span><span class="o">.</span><span class="n">text</span> <span class="o">=</span> <span class="n">todos</span><span class="p">[</span><span class="n">indexPath</span><span class="o">.</span><span class="n">row</span><span class="p">]</span>
            <span class="k">return</span> <span class="n">cell</span>
        <span class="k">default</span><span class="p">:</span>
            <span class="nf">fatalError</span><span class="p">()</span>
        <span class="p">}</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

<p><code class="highlighter-rouge">.todos</code> 的情况下很简单，我们就用标准的 <code class="highlighter-rouge">UITableViewCell</code> 就好。对于 <code class="highlighter-rouge">.input</code> 的情况，我们需要在 cell 里嵌一个 <code class="highlighter-rouge">UITextField</code>，并且要在其中的文本改变时能告知 <code class="highlighter-rouge">TableViewController</code>。我们可以使用传统的 delegate 的模式来实现，下面是 TableViewInputCell.swift 的内容：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">protocol</span> <span class="kt">TableViewInputCellDelegate</span><span class="p">:</span> <span class="kd">class</span> <span class="p">{</span>
    <span class="kd">func</span> <span class="nf">inputChanged</span><span class="p">(</span><span class="nv">cell</span><span class="p">:</span> <span class="kt">TableViewInputCell</span><span class="p">,</span> <span class="nv">text</span><span class="p">:</span> <span class="kt">String</span><span class="p">)</span>
<span class="p">}</span>

<span class="kd">class</span> <span class="kt">TableViewInputCell</span><span class="p">:</span> <span class="kt">UITableViewCell</span> <span class="p">{</span>
    <span class="k">weak</span> <span class="k">var</span> <span class="nv">delegate</span><span class="p">:</span> <span class="kt">TableViewInputCellDelegate</span><span class="p">?</span>
    <span class="kd">@IBOutlet</span> <span class="k">weak</span> <span class="k">var</span> <span class="nv">textField</span><span class="p">:</span> <span class="kt">UITextField</span><span class="o">!</span>
    
    <span class="kd">@objc</span> <span class="kd">@IBAction</span> <span class="kd">func</span> <span class="nf">textFieldValueChanged</span><span class="p">(</span><span class="n">_</span> <span class="nv">sender</span><span class="p">:</span> <span class="kt">UITextField</span><span class="p">)</span> <span class="p">{</span>
        <span class="n">delegate</span><span class="p">?</span><span class="o">.</span><span class="nf">inputChanged</span><span class="p">(</span><span class="nv">cell</span><span class="p">:</span> <span class="k">self</span><span class="p">,</span> <span class="nv">text</span><span class="p">:</span> <span class="n">sender</span><span class="o">.</span><span class="n">text</span> <span class="p">??</span> <span class="s">""</span><span class="p">)</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

<p>我们在 Storyboard 中创建对应的 table view 和这个 cell，然后将其中的 text field 的 <code class="highlighter-rouge">.editingChanged</code> 事件绑到 <code class="highlighter-rouge">textFieldValueChanged</code> 上。每次当用户进行输入时，<code class="highlighter-rouge">delegate</code> 的方法将被调用。</p>

<p>在 <code class="highlighter-rouge">TableViewController</code> 里，现在可以返回 <code class="highlighter-rouge">.input</code> 的 cell，并设置对应的代理方法来更新添加按钮了：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code>
<span class="kd">class</span> <span class="kt">TableViewController</span><span class="p">:</span> <span class="kt">UITableViewController</span> <span class="p">{</span>
    <span class="k">override</span> <span class="kd">func</span> <span class="nf">tableView</span><span class="p">(</span><span class="n">_</span> <span class="nv">tableView</span><span class="p">:</span> <span class="kt">UITableView</span><span class="p">,</span> <span class="n">cellForRowAt</span> <span class="nv">indexPath</span><span class="p">:</span> <span class="kt">IndexPath</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="kt">UITableViewCell</span> <span class="p">{</span>
        <span class="k">guard</span> <span class="k">let</span> <span class="nv">section</span> <span class="o">=</span> <span class="kt">Section</span><span class="p">(</span><span class="nv">rawValue</span><span class="p">:</span> <span class="n">indexPath</span><span class="o">.</span><span class="n">section</span><span class="p">)</span> <span class="k">else</span> <span class="p">{</span>
            <span class="nf">fatalError</span><span class="p">()</span>
        <span class="p">}</span>
        
        <span class="k">switch</span> <span class="n">section</span> <span class="p">{</span>
        <span class="k">case</span> <span class="o">.</span><span class="nv">input</span><span class="p">:</span>
            <span class="k">let</span> <span class="nv">cell</span> <span class="o">=</span> <span class="n">tableView</span><span class="o">.</span><span class="nf">dequeueReusableCell</span><span class="p">(</span><span class="nv">withIdentifier</span><span class="p">:</span> <span class="n">inputCellReuseId</span><span class="p">,</span> <span class="nv">for</span><span class="p">:</span> <span class="n">indexPath</span><span class="p">)</span> <span class="k">as!</span> <span class="kt">TableViewInputCell</span>
            <span class="n">cell</span><span class="o">.</span><span class="n">delegate</span> <span class="o">=</span> <span class="k">self</span>
            <span class="k">return</span> <span class="n">cell</span>
        <span class="c1">//...</span>
        <span class="p">}</span>
    <span class="p">}</span>
<span class="p">}</span>

<span class="kd">extension</span> <span class="kt">TableViewController</span><span class="p">:</span> <span class="kt">TableViewInputCellDelegate</span> <span class="p">{</span>
    <span class="kd">func</span> <span class="nf">inputChanged</span><span class="p">(</span><span class="nv">cell</span><span class="p">:</span> <span class="kt">TableViewInputCell</span><span class="p">,</span> <span class="nv">text</span><span class="p">:</span> <span class="kt">String</span><span class="p">)</span> <span class="p">{</span>
        <span class="k">let</span> <span class="nv">isItemLengthEnough</span> <span class="o">=</span> <span class="n">text</span><span class="o">.</span><span class="n">count</span> <span class="o">&gt;=</span> <span class="mi">3</span>
        <span class="n">navigationItem</span><span class="o">.</span><span class="n">rightBarButtonItem</span><span class="p">?</span><span class="o">.</span><span class="n">isEnabled</span> <span class="o">=</span> <span class="n">isItemLengthEnough</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

<p>现在，运行程序后等待一段时间，读入的待办事项就可以被展示了。接下来，添加待办和移除待办的部分很容易实现：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">class</span> <span class="kt">TableViewController</span><span class="p">:</span> <span class="kt">UITableViewController</span> <span class="p">{</span>
    <span class="c1">// 添加待办</span>
    <span class="kd">@IBAction</span> <span class="kd">func</span> <span class="nf">addButtonPressed</span><span class="p">(</span><span class="n">_</span> <span class="nv">sender</span><span class="p">:</span> <span class="kt">Any</span><span class="p">)</span> <span class="p">{</span>
        <span class="k">let</span> <span class="nv">inputIndexPath</span> <span class="o">=</span> <span class="kt">IndexPath</span><span class="p">(</span><span class="nv">row</span><span class="p">:</span> <span class="mi">0</span><span class="p">,</span> <span class="nv">section</span><span class="p">:</span> <span class="kt">Section</span><span class="o">.</span><span class="n">input</span><span class="o">.</span><span class="n">rawValue</span><span class="p">)</span>
        <span class="k">guard</span> <span class="k">let</span> <span class="nv">inputCell</span> <span class="o">=</span> <span class="n">tableView</span><span class="o">.</span><span class="nf">cellForRow</span><span class="p">(</span><span class="nv">at</span><span class="p">:</span> <span class="n">inputIndexPath</span><span class="p">)</span> <span class="k">as?</span> <span class="kt">TableViewInputCell</span><span class="p">,</span>
              <span class="k">let</span> <span class="nv">text</span> <span class="o">=</span> <span class="n">inputCell</span><span class="o">.</span><span class="n">textField</span><span class="o">.</span><span class="n">text</span> <span class="k">else</span>
        <span class="p">{</span>
            <span class="k">return</span>
        <span class="p">}</span>
        <span class="n">todos</span><span class="o">.</span><span class="nf">insert</span><span class="p">(</span><span class="n">text</span><span class="p">,</span> <span class="nv">at</span><span class="p">:</span> <span class="mi">0</span><span class="p">)</span>
        <span class="n">inputCell</span><span class="o">.</span><span class="n">textField</span><span class="o">.</span><span class="n">text</span> <span class="o">=</span> <span class="s">""</span>
        <span class="n">title</span> <span class="o">=</span> <span class="s">"TODO - (</span><span class="se">\(</span><span class="n">todos</span><span class="o">.</span><span class="n">count</span><span class="se">)</span><span class="s">)"</span>
        <span class="n">tableView</span><span class="o">.</span><span class="nf">reloadData</span><span class="p">()</span>
    <span class="p">}</span>

    <span class="c1">// 移除待办</span>
    <span class="k">override</span> <span class="kd">func</span> <span class="nf">tableView</span><span class="p">(</span><span class="n">_</span> <span class="nv">tableView</span><span class="p">:</span> <span class="kt">UITableView</span><span class="p">,</span> <span class="n">didSelectRowAt</span> <span class="nv">indexPath</span><span class="p">:</span> <span class="kt">IndexPath</span><span class="p">)</span> <span class="p">{</span>
        <span class="k">guard</span> <span class="n">indexPath</span><span class="o">.</span><span class="n">section</span> <span class="o">==</span> <span class="kt">Section</span><span class="o">.</span><span class="n">todos</span><span class="o">.</span><span class="n">rawValue</span> <span class="k">else</span> <span class="p">{</span>
            <span class="k">return</span>
        <span class="p">}</span>
        
        <span class="n">todos</span><span class="o">.</span><span class="nf">remove</span><span class="p">(</span><span class="nv">at</span><span class="p">:</span> <span class="n">indexPath</span><span class="o">.</span><span class="n">row</span><span class="p">)</span>
        <span class="n">title</span> <span class="o">=</span> <span class="s">"TODO - (</span><span class="se">\(</span><span class="n">todos</span><span class="o">.</span><span class="n">count</span><span class="se">)</span><span class="s">)"</span>
        <span class="n">tableView</span><span class="o">.</span><span class="nf">reloadData</span><span class="p">()</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

<blockquote>
  <p>为了保持简单，这里我们直接 <code class="highlighter-rouge">tableView.reloadData()</code> 了，讲道理的话更好的选择是只针对变动的部分做 <code class="highlighter-rouge">insert</code> 或者 <code class="highlighter-rouge">remove</code>，但为了简单起见，我们就直接重载整个 table view 了。</p>
</blockquote>

<p>好了，这是一个非常简单的一百行都不到的 View Controller，可能也是我们每天都会写的代码，所以我们就不吹捧这样的代码“条理清晰”或者“简洁明了”了，你我都知道这只是在 View Controller 规模尚小时的假象而已。让我们直接来看看潜在的问题：</p>

<ol>
  <li>UI 相关的代码散落各处 - 重载 <code class="highlighter-rouge">tableView</code> 和设置 <code class="highlighter-rouge">title</code> 的代码出现了三次，设置右上 button 的 <code class="highlighter-rouge">isEnabled</code> 的代码存在于 extension 中，添加新项目时我们先获取了输入的 cell，然后再读取 cell 中的文本。这些散落在各处的 UI 操作将会成为隐患，因为你可能在代码的任意部分操作这些 UI，而它们的状态将随着代码的复杂变得“飘忽不定”。</li>
  <li>因为 1 的状态复杂，使得 View Controller 难以测试 - 举个例子，如果你想测试 <code class="highlighter-rouge">title</code> 的文字正确，你可能需要手动向列表里添加一个待办事项，这涉及到调用 <code class="highlighter-rouge">addButtonPressed</code>，而这个方法需要读取 <code class="highlighter-rouge">inputCell</code> 的文本，那么你可能还需要先去设置这个 cell 中 <code class="highlighter-rouge">UITextField</code> 的 <code class="highlighter-rouge">text</code> 值。当然你也可以用依赖注入的方式给 <code class="highlighter-rouge">add</code> 方法一个文本参数，或者将 <code class="highlighter-rouge">todos.insert</code> 和之后的内容提取成一个新的方法，但是无论怎么样，对于 model 的操作和对于 UI 的更新都没有分离 (因为毕竟我们写的就是“胶水代码”)。这正是你觉得 View Controller 难以测试的最主要原因。</li>
  <li>因为 2 的难以测试，最后让 View Controller 难以重构 - 状态和 UI 复杂度的增加往往会导致多个 UI 操作维护着同一个变量，或者多个状态变量去更新同一个 UI 元素。不论是哪种情况，都让测试变得几乎不可能，也会让后续的开发人员 (其实往往就是你自己！) 在面对复杂情况下难以正确地继续开发。Massive View Controller 最终的结果常常是牵一发而动全身，一个微小的改动可能都需要花费大量的时间进行验证，而且还没有人敢拍胸脯保证正确性。这会让项目逐渐陷入泥潭。</li>
</ol>

<p>这些问题最终导致，这样一个 View Controller 难以 scaling。在逐渐被代码填满到一两千行时，这个 View Controller 将彻底“死去”，对它的维护和更改会困难重重。</p>

<blockquote>
  <p>你可以在 GitHub repo 的 <a href="https://github.com/onevcat/ToDoDemo/tree/basic">basic 分支</a>找到对应这部分的代码。</p>
</blockquote>

<h2 id="基于-state-的-view-controller">基于 State 的 View Controller</h2>

<h3 id="通过提取-state-统合-ui-操作">通过提取 State 统合 UI 操作</h3>

<p>上面的三个问题其实环环相扣，如果我们能将 UI 相关代码集中起来，并用单一的状态去管理它，就可以让 View Controller 的复杂度降低很多。我们尝试看看！</p>

<p>在这个简单的界面中，和 UI 相关的 model 包括待办条目 <code class="highlighter-rouge">todos</code> (用来组织 table view 和更新标题栏) 以及输入的 <code class="highlighter-rouge">text</code> (用来决定添加按钮的 enable 和添加 todo 时的内容)。我们将这两个变量进行简单的封装，在 <code class="highlighter-rouge">TableViewController</code> 里添加一个内嵌的 <code class="highlighter-rouge">State</code> 结构体：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">class</span> <span class="kt">TableViewController</span><span class="p">:</span> <span class="kt">UITableViewController</span> <span class="p">{</span>
    
    <span class="kd">struct</span> <span class="kt">State</span> <span class="p">{</span>
        <span class="k">let</span> <span class="nv">todos</span><span class="p">:</span> <span class="p">[</span><span class="kt">String</span><span class="p">]</span>
        <span class="k">let</span> <span class="nv">text</span><span class="p">:</span> <span class="kt">String</span>
    <span class="p">}</span>
    
    <span class="k">var</span> <span class="nv">state</span> <span class="o">=</span> <span class="kt">State</span><span class="p">(</span><span class="nv">todos</span><span class="p">:</span> <span class="p">[],</span> <span class="nv">text</span><span class="p">:</span> <span class="s">""</span><span class="p">)</span>
<span class="p">}</span>
</code></pre></div></div>

<p>这样一来，我们就有一个统一按照状态更新 UI 的地方了。使用 <code class="highlighter-rouge">state</code> 的 <code class="highlighter-rouge">didSet</code> 即可：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">var</span> <span class="nv">state</span> <span class="o">=</span> <span class="kt">State</span><span class="p">(</span><span class="nv">todos</span><span class="p">:</span> <span class="p">[],</span> <span class="nv">text</span><span class="p">:</span> <span class="s">""</span><span class="p">)</span> <span class="p">{</span>
     <span class="k">didSet</span> <span class="p">{</span>
        <span class="k">if</span> <span class="n">oldValue</span><span class="o">.</span><span class="n">todos</span> <span class="o">!=</span> <span class="n">state</span><span class="o">.</span><span class="n">todos</span> <span class="p">{</span>
            <span class="n">tableView</span><span class="o">.</span><span class="nf">reloadData</span><span class="p">()</span>
            <span class="n">title</span> <span class="o">=</span> <span class="s">"TODO - (</span><span class="se">\(</span><span class="n">state</span><span class="o">.</span><span class="n">todos</span><span class="o">.</span><span class="n">count</span><span class="se">)</span><span class="s">)"</span>
        <span class="p">}</span>

        <span class="k">if</span> <span class="p">(</span><span class="n">oldValue</span><span class="o">.</span><span class="n">text</span> <span class="o">!=</span> <span class="n">state</span><span class="o">.</span><span class="n">text</span><span class="p">)</span> <span class="p">{</span>
            <span class="k">let</span> <span class="nv">isItemLengthEnough</span> <span class="o">=</span> <span class="n">state</span><span class="o">.</span><span class="n">text</span><span class="o">.</span><span class="n">count</span> <span class="o">&gt;=</span> <span class="mi">3</span>
            <span class="n">navigationItem</span><span class="o">.</span><span class="n">rightBarButtonItem</span><span class="p">?</span><span class="o">.</span><span class="n">isEnabled</span> <span class="o">=</span> <span class="n">isItemLengthEnough</span>

            <span class="k">let</span> <span class="nv">inputIndexPath</span> <span class="o">=</span> <span class="kt">IndexPath</span><span class="p">(</span><span class="nv">row</span><span class="p">:</span> <span class="mi">0</span><span class="p">,</span> <span class="nv">section</span><span class="p">:</span> <span class="kt">Section</span><span class="o">.</span><span class="n">input</span><span class="o">.</span><span class="n">rawValue</span><span class="p">)</span>
            <span class="k">let</span> <span class="nv">inputCell</span> <span class="o">=</span> <span class="n">tableView</span><span class="o">.</span><span class="nf">cellForRow</span><span class="p">(</span><span class="nv">at</span><span class="p">:</span> <span class="n">inputIndexPath</span><span class="p">)</span> <span class="k">as?</span> <span class="kt">TableViewInputCell</span>
            <span class="n">inputCell</span><span class="p">?</span><span class="o">.</span><span class="n">textField</span><span class="o">.</span><span class="n">text</span> <span class="o">=</span> <span class="n">state</span><span class="o">.</span><span class="n">text</span>
        <span class="p">}</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

<p>这里我们将新值和旧值进行了一些比较，以避免不必要的 UI 更新。接下来，就可以将原来 <code class="highlighter-rouge">TableViewController</code> 中对 UI 的操作换成对 <code class="highlighter-rouge">state</code> 的操作了。</p>

<p>比如，在 <code class="highlighter-rouge">viewDidLoad</code> 中：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1">// 变更前</span>
<span class="kt">ToDoStore</span><span class="o">.</span><span class="n">shared</span><span class="o">.</span><span class="n">getToDoItems</span> <span class="p">{</span> <span class="p">(</span><span class="n">data</span><span class="p">)</span> <span class="k">in</span>
    <span class="k">self</span><span class="o">.</span><span class="n">todos</span> <span class="o">+=</span> <span class="n">data</span>
    <span class="k">self</span><span class="o">.</span><span class="n">title</span> <span class="o">=</span> <span class="s">"TODO - (</span><span class="se">\(</span><span class="k">self</span><span class="o">.</span><span class="n">todos</span><span class="o">.</span><span class="n">count</span><span class="se">)</span><span class="s">)"</span>
    <span class="k">self</span><span class="o">.</span><span class="n">tableView</span><span class="o">.</span><span class="nf">reloadData</span><span class="p">()</span>
<span class="p">}</span>

<span class="c1">// 变更后</span>
<span class="kt">ToDoStore</span><span class="o">.</span><span class="n">shared</span><span class="o">.</span><span class="n">getToDoItems</span> <span class="p">{</span> <span class="p">(</span><span class="n">data</span><span class="p">)</span> <span class="k">in</span>
    <span class="k">self</span><span class="o">.</span><span class="n">state</span> <span class="o">=</span> <span class="kt">State</span><span class="p">(</span><span class="nv">todos</span><span class="p">:</span> <span class="k">self</span><span class="o">.</span><span class="n">state</span><span class="o">.</span><span class="n">todos</span> <span class="o">+</span> <span class="n">data</span><span class="p">,</span> <span class="nv">text</span><span class="p">:</span> <span class="k">self</span><span class="o">.</span><span class="n">state</span><span class="o">.</span><span class="n">text</span><span class="p">)</span>
<span class="p">}</span>
</code></pre></div></div>

<p>点击 cell 移除待办时：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1">// 变更前</span>
<span class="k">override</span> <span class="kd">func</span> <span class="nf">tableView</span><span class="p">(</span><span class="n">_</span> <span class="nv">tableView</span><span class="p">:</span> <span class="kt">UITableView</span><span class="p">,</span> <span class="n">didSelectRowAt</span> <span class="nv">indexPath</span><span class="p">:</span> <span class="kt">IndexPath</span><span class="p">)</span> <span class="p">{</span>
   <span class="k">guard</span> <span class="n">indexPath</span><span class="o">.</span><span class="n">section</span> <span class="o">==</span> <span class="kt">Section</span><span class="o">.</span><span class="n">todos</span><span class="o">.</span><span class="n">rawValue</span> <span class="k">else</span> <span class="p">{</span>
       <span class="k">return</span>
   <span class="p">}</span>
   
   <span class="n">todos</span><span class="o">.</span><span class="nf">remove</span><span class="p">(</span><span class="nv">at</span><span class="p">:</span> <span class="n">indexPath</span><span class="o">.</span><span class="n">row</span><span class="p">)</span>
   <span class="n">title</span> <span class="o">=</span> <span class="s">"TODO - (</span><span class="se">\(</span><span class="n">todos</span><span class="o">.</span><span class="n">count</span><span class="se">)</span><span class="s">)"</span>
   <span class="n">tableView</span><span class="o">.</span><span class="nf">reloadData</span><span class="p">()</span>
<span class="p">}</span>

<span class="c1">// 变更后</span>
<span class="k">override</span> <span class="kd">func</span> <span class="nf">tableView</span><span class="p">(</span><span class="n">_</span> <span class="nv">tableView</span><span class="p">:</span> <span class="kt">UITableView</span><span class="p">,</span> <span class="n">didSelectRowAt</span> <span class="nv">indexPath</span><span class="p">:</span> <span class="kt">IndexPath</span><span class="p">)</span> <span class="p">{</span>
     <span class="k">guard</span> <span class="n">indexPath</span><span class="o">.</span><span class="n">section</span> <span class="o">==</span> <span class="kt">Section</span><span class="o">.</span><span class="n">todos</span><span class="o">.</span><span class="n">rawValue</span> <span class="k">else</span> <span class="p">{</span>
        <span class="k">return</span>
     <span class="p">}</span>
   
     <span class="k">let</span> <span class="nv">newTodos</span> <span class="o">=</span> <span class="kt">Array</span><span class="p">(</span><span class="n">state</span><span class="o">.</span><span class="n">todos</span><span class="p">[</span><span class="o">..&lt;</span><span class="n">indexPath</span><span class="o">.</span><span class="n">row</span><span class="p">]</span> <span class="o">+</span> <span class="n">state</span><span class="o">.</span><span class="n">todos</span><span class="p">[(</span><span class="n">indexPath</span><span class="o">.</span><span class="n">row</span> <span class="o">+</span> <span class="mi">1</span><span class="p">)</span><span class="o">...</span><span class="p">])</span>
     <span class="n">state</span> <span class="o">=</span> <span class="kt">State</span><span class="p">(</span><span class="nv">todos</span><span class="p">:</span> <span class="n">newTodos</span><span class="p">,</span> <span class="nv">text</span><span class="p">:</span> <span class="n">state</span><span class="o">.</span><span class="n">text</span><span class="p">)</span>
<span class="p">}</span>
</code></pre></div></div>

<p>在输入框键入文字时：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1">// 变更前</span>
<span class="kd">func</span> <span class="nf">inputChanged</span><span class="p">(</span><span class="nv">cell</span><span class="p">:</span> <span class="kt">TableViewInputCell</span><span class="p">,</span> <span class="nv">text</span><span class="p">:</span> <span class="kt">String</span><span class="p">)</span> <span class="p">{</span>
     <span class="k">let</span> <span class="nv">isItemLengthEnough</span> <span class="o">=</span> <span class="n">text</span><span class="o">.</span><span class="n">count</span> <span class="o">&gt;=</span> <span class="mi">3</span>
     <span class="n">navigationItem</span><span class="o">.</span><span class="n">rightBarButtonItem</span><span class="p">?</span><span class="o">.</span><span class="n">isEnabled</span> <span class="o">=</span> <span class="n">isItemLengthEnough</span>
<span class="p">}</span>

<span class="c1">// 变更后</span>
<span class="kd">func</span> <span class="nf">inputChanged</span><span class="p">(</span><span class="nv">cell</span><span class="p">:</span> <span class="kt">TableViewInputCell</span><span class="p">,</span> <span class="nv">text</span><span class="p">:</span> <span class="kt">String</span><span class="p">)</span> <span class="p">{</span>
   <span class="n">state</span> <span class="o">=</span> <span class="kt">State</span><span class="p">(</span><span class="nv">todos</span><span class="p">:</span> <span class="n">state</span><span class="o">.</span><span class="n">todos</span><span class="p">,</span> <span class="nv">text</span><span class="p">:</span> <span class="n">text</span><span class="p">)</span>
<span class="p">}</span>
</code></pre></div></div>

<p>另外，最值得一提的可能是添加待办事项时的代码变化。可以看到引入统一的状态变更后，代码变得非常简单清晰：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1">// 变更前</span>
<span class="kd">@IBAction</span> <span class="kd">func</span> <span class="nf">addButtonPressed</span><span class="p">(</span><span class="n">_</span> <span class="nv">sender</span><span class="p">:</span> <span class="kt">Any</span><span class="p">)</span> <span class="p">{</span>
    <span class="k">let</span> <span class="nv">inputIndexPath</span> <span class="o">=</span> <span class="kt">IndexPath</span><span class="p">(</span><span class="nv">row</span><span class="p">:</span> <span class="mi">0</span><span class="p">,</span> <span class="nv">section</span><span class="p">:</span> <span class="kt">Section</span><span class="o">.</span><span class="n">input</span><span class="o">.</span><span class="n">rawValue</span><span class="p">)</span>
    <span class="k">guard</span> <span class="k">let</span> <span class="nv">inputCell</span> <span class="o">=</span> <span class="n">tableView</span><span class="o">.</span><span class="nf">cellForRow</span><span class="p">(</span><span class="nv">at</span><span class="p">:</span> <span class="n">inputIndexPath</span><span class="p">)</span> <span class="k">as?</span> <span class="kt">TableViewInputCell</span><span class="p">,</span>
          <span class="k">let</span> <span class="nv">text</span> <span class="o">=</span> <span class="n">inputCell</span><span class="o">.</span><span class="n">textField</span><span class="o">.</span><span class="n">text</span> <span class="k">else</span>
    <span class="p">{</span>
        <span class="k">return</span>
    <span class="p">}</span>
    <span class="n">todos</span><span class="o">.</span><span class="nf">insert</span><span class="p">(</span><span class="n">text</span><span class="p">,</span> <span class="nv">at</span><span class="p">:</span> <span class="mi">0</span><span class="p">)</span>
    <span class="n">inputCell</span><span class="o">.</span><span class="n">textField</span><span class="o">.</span><span class="n">text</span> <span class="o">=</span> <span class="s">""</span>
    <span class="n">title</span> <span class="o">=</span> <span class="s">"TODO - (</span><span class="se">\(</span><span class="n">todos</span><span class="o">.</span><span class="n">count</span><span class="se">)</span><span class="s">)"</span>
    <span class="n">tableView</span><span class="o">.</span><span class="nf">reloadData</span><span class="p">()</span>
<span class="p">}</span>

<span class="c1">// 变更后</span>
<span class="kd">@IBAction</span> <span class="kd">func</span> <span class="nf">addButtonPressed</span><span class="p">(</span><span class="n">_</span> <span class="nv">sender</span><span class="p">:</span> <span class="kt">Any</span><span class="p">)</span> <span class="p">{</span>
    <span class="n">state</span> <span class="o">=</span> <span class="kt">State</span><span class="p">(</span><span class="nv">todos</span><span class="p">:</span> <span class="p">[</span><span class="n">state</span><span class="o">.</span><span class="n">text</span><span class="p">]</span> <span class="o">+</span> <span class="n">state</span><span class="o">.</span><span class="n">todos</span><span class="p">,</span> <span class="nv">text</span><span class="p">:</span> <span class="s">""</span><span class="p">)</span>
<span class="p">}</span>
</code></pre></div></div>

<blockquote>
  <p>如果你对 React 比较熟悉的话，可以从中发现一些类似的思想。React 里我们自上而下传递 <code class="highlighter-rouge">Props</code>，并且在 Component 自身通过 <code class="highlighter-rouge">setState</code> 进行状态管理。所有的 <code class="highlighter-rouge">Component</code> 都是基于传入的 <code class="highlighter-rouge">Props</code> 和自身的 <code class="highlighter-rouge">State</code> 的。View Controller 中的不同之处在于，React 使用了更为描述式的方式更新 UI (虚拟 DOM)，而现在我们可能需要用过程语言自己进行实现。除此之外，使用 <code class="highlighter-rouge">State</code> 的 <code class="highlighter-rouge">TableViewController</code> 在工作方式上与 React 的 <code class="highlighter-rouge">Component</code> 十分类似。</p>
</blockquote>

<h3 id="测试-state-view-controller">测试 State View Controller</h3>

<p>在基于 <code class="highlighter-rouge">State</code> 的实现下，用户的操作被统一为状态的变更，而状态的变更将统一地去更新当前的 UI。这让 View Controller 的测试变得容易很多。我们可以将本来混杂在一起的行为分离开来：首先，测试状态变更可以导致正确的 UI；然后，测试用户输入可以导致正确的状态变更，这样即可覆盖 View Controller 的测试。</p>

<p>让我们先来测试状态变更导致的 UI 变化，在单元测试中：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">func</span> <span class="nf">testSettingState</span><span class="p">()</span> <span class="p">{</span>
    <span class="c1">// 初始状态</span>
    <span class="kt">XCTAssertEqual</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">tableView</span><span class="o">.</span><span class="nf">numberOfRows</span><span class="p">(</span><span class="nv">inSection</span><span class="p">:</span> <span class="kt">TableViewController</span><span class="o">.</span><span class="kt">Section</span><span class="o">.</span><span class="n">todos</span><span class="o">.</span><span class="n">rawValue</span><span class="p">),</span> <span class="mi">0</span><span class="p">)</span>
    <span class="kt">XCTAssertEqual</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">title</span><span class="p">,</span> <span class="s">"TODO - (0)"</span><span class="p">)</span>
    <span class="kt">XCTAssertFalse</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">navigationItem</span><span class="o">.</span><span class="n">rightBarButtonItem</span><span class="o">!.</span><span class="n">isEnabled</span><span class="p">)</span>

    <span class="c1">// ([], "") -&gt; (["1", "2", "3"], "abc")</span>
    <span class="n">controller</span><span class="o">.</span><span class="n">state</span> <span class="o">=</span> <span class="kt">TableViewController</span><span class="o">.</span><span class="kt">State</span><span class="p">(</span><span class="nv">todos</span><span class="p">:</span> <span class="p">[</span><span class="s">"1"</span><span class="p">,</span> <span class="s">"2"</span><span class="p">,</span> <span class="s">"3"</span><span class="p">],</span> <span class="nv">text</span><span class="p">:</span> <span class="s">"abc"</span><span class="p">)</span>
    <span class="kt">XCTAssertEqual</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">tableView</span><span class="o">.</span><span class="nf">numberOfRows</span><span class="p">(</span><span class="nv">inSection</span><span class="p">:</span> <span class="kt">TableViewController</span><span class="o">.</span><span class="kt">Section</span><span class="o">.</span><span class="n">todos</span><span class="o">.</span><span class="n">rawValue</span><span class="p">),</span> <span class="mi">3</span><span class="p">)</span>
    <span class="kt">XCTAssertEqual</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">tableView</span><span class="o">.</span><span class="nf">cellForRow</span><span class="p">(</span><span class="nv">at</span><span class="p">:</span> <span class="nf">todoItemIndexPath</span><span class="p">(</span><span class="nv">row</span><span class="p">:</span> <span class="mi">1</span><span class="p">))?</span><span class="o">.</span><span class="n">textLabel</span><span class="p">?</span><span class="o">.</span><span class="n">text</span><span class="p">,</span> <span class="s">"2"</span><span class="p">)</span>
    <span class="kt">XCTAssertEqual</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">title</span><span class="p">,</span> <span class="s">"TODO - (3)"</span><span class="p">)</span>
    <span class="kt">XCTAssertTrue</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">navigationItem</span><span class="o">.</span><span class="n">rightBarButtonItem</span><span class="o">!.</span><span class="n">isEnabled</span><span class="p">)</span>

    <span class="c1">// (["1", "2", "3"], "abc") -&gt; ([], "")</span>
    <span class="n">controller</span><span class="o">.</span><span class="n">state</span> <span class="o">=</span> <span class="kt">TableViewController</span><span class="o">.</span><span class="kt">State</span><span class="p">(</span><span class="nv">todos</span><span class="p">:</span> <span class="p">[],</span> <span class="nv">text</span><span class="p">:</span> <span class="s">""</span><span class="p">)</span>
    <span class="kt">XCTAssertEqual</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">tableView</span><span class="o">.</span><span class="nf">numberOfRows</span><span class="p">(</span><span class="nv">inSection</span><span class="p">:</span> <span class="kt">TableViewController</span><span class="o">.</span><span class="kt">Section</span><span class="o">.</span><span class="n">todos</span><span class="o">.</span><span class="n">rawValue</span><span class="p">),</span> <span class="mi">0</span><span class="p">)</span>
    <span class="kt">XCTAssertEqual</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">title</span><span class="p">,</span> <span class="s">"TODO - (0)"</span><span class="p">)</span>
    <span class="kt">XCTAssertFalse</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">navigationItem</span><span class="o">.</span><span class="n">rightBarButtonItem</span><span class="o">!.</span><span class="n">isEnabled</span><span class="p">)</span>
<span class="p">}</span>
</code></pre></div></div>

<p>这里的初始状态是我们在 Storyboard 或者相应的 <code class="highlighter-rouge">viewDidLoad</code> 之类的方法里设定的 UI。我们稍后会对这个状态进行进一步的讨论。</p>

<p>接下来，我们就可以测试用户的交互行为导致的状态变更了：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">func</span> <span class="nf">testAdding</span><span class="p">()</span> <span class="p">{</span>
    <span class="k">let</span> <span class="nv">testItem</span> <span class="o">=</span> <span class="s">"Test Item"</span>

    <span class="k">let</span> <span class="nv">originalTodos</span> <span class="o">=</span> <span class="n">controller</span><span class="o">.</span><span class="n">state</span><span class="o">.</span><span class="n">todos</span>
    <span class="n">controller</span><span class="o">.</span><span class="n">state</span> <span class="o">=</span> <span class="kt">TableViewController</span><span class="o">.</span><span class="kt">State</span><span class="p">(</span><span class="nv">todos</span><span class="p">:</span> <span class="n">originalTodos</span><span class="p">,</span> <span class="nv">text</span><span class="p">:</span> <span class="n">testItem</span><span class="p">)</span>
    <span class="n">controller</span><span class="o">.</span><span class="nf">addButtonPressed</span><span class="p">(</span><span class="k">self</span><span class="p">)</span>
    <span class="kt">XCTAssertEqual</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">state</span><span class="o">.</span><span class="n">todos</span><span class="p">,</span> <span class="p">[</span><span class="n">testItem</span><span class="p">]</span> <span class="o">+</span> <span class="n">originalTodos</span><span class="p">)</span>
    <span class="kt">XCTAssertEqual</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">state</span><span class="o">.</span><span class="n">text</span><span class="p">,</span> <span class="s">""</span><span class="p">)</span>
<span class="p">}</span>
    
<span class="kd">func</span> <span class="nf">testRemoving</span><span class="p">()</span> <span class="p">{</span>
    <span class="n">controller</span><span class="o">.</span><span class="n">state</span> <span class="o">=</span> <span class="kt">TableViewController</span><span class="o">.</span><span class="kt">State</span><span class="p">(</span><span class="nv">todos</span><span class="p">:</span> <span class="p">[</span><span class="s">"1"</span><span class="p">,</span> <span class="s">"2"</span><span class="p">,</span> <span class="s">"3"</span><span class="p">],</span> <span class="nv">text</span><span class="p">:</span> <span class="s">""</span><span class="p">)</span>
    <span class="n">controller</span><span class="o">.</span><span class="nf">tableView</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">tableView</span><span class="p">,</span> <span class="nv">didSelectRowAt</span><span class="p">:</span> <span class="nf">todoItemIndexPath</span><span class="p">(</span><span class="nv">row</span><span class="p">:</span> <span class="mi">1</span><span class="p">))</span>
    <span class="kt">XCTAssertEqual</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">state</span><span class="o">.</span><span class="n">todos</span><span class="p">,</span> <span class="p">[</span><span class="s">"1"</span><span class="p">,</span> <span class="s">"3"</span><span class="p">])</span>
<span class="p">}</span>
    
<span class="kd">func</span> <span class="nf">testInputChanged</span><span class="p">()</span> <span class="p">{</span>
    <span class="n">controller</span><span class="o">.</span><span class="nf">inputChanged</span><span class="p">(</span><span class="nv">cell</span><span class="p">:</span> <span class="kt">TableViewInputCell</span><span class="p">(),</span> <span class="nv">text</span><span class="p">:</span> <span class="s">"Hello"</span><span class="p">)</span>
    <span class="kt">XCTAssertEqual</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">state</span><span class="o">.</span><span class="n">text</span><span class="p">,</span> <span class="s">"Hello"</span><span class="p">)</span>
<span class="p">}</span>
</code></pre></div></div>

<p>看起来很赞，我们的单元测试覆盖了各种用户交互，配合上 state 变更导致的 UI 变化，我们几乎可以确定这个 View Controller 将会按照我们的设想正确工作了！</p>

<blockquote>
  <p>在上面我只贴出了一些关键的变更，关于测试的配置以及一些其他细节，你可以参看 GitHub repo 的 <a href="https://github.com/onevcat/ToDoDemo/tree/state">state 分支</a>。</p>
</blockquote>

<h3 id="state-view-controller-的问题">State View Controller 的问题</h3>

<p>这种基于 State 的 View Controller 虽然比原来好了很多，但是依然存在一些问题，也还有大量的改进空间。下面是几个主要的忧虑：</p>

<ol>
  <li>
    <p>初始化时的 UI - 我们上面说到过，初始状态的 UI 是我们在 Storyboard 或者相应的 <code class="highlighter-rouge">viewDidLoad</code> 之类的方法里设定的。这将导致一个问题，那就是我们无法通过设置 <code class="highlighter-rouge">state</code> 属性的方式来设置初始 UI。因为 <code class="highlighter-rouge">state</code> 的 <code class="highlighter-rouge">didSet</code> 不会在 controller 初始化中首次赋值时被调用，因此如果我们在 <code class="highlighter-rouge">viewDidLoad</code> 中添加如下语句的话，会因为新的状态和初始相同，而导致 UI 不发生更新：</p>

    <div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code> <span class="k">override</span> <span class="kd">func</span> <span class="nf">viewDidLoad</span><span class="p">()</span> <span class="p">{</span>
     <span class="k">super</span><span class="o">.</span><span class="nf">viewDidLoad</span><span class="p">()</span>
        
     <span class="c1">// UI 更新会被跳过，因为该 state 和初始值的一样</span>
     <span class="n">state</span> <span class="o">=</span> <span class="kt">State</span><span class="p">(</span><span class="nv">todos</span><span class="p">:</span> <span class="p">[],</span> <span class="nv">text</span><span class="p">:</span> <span class="s">""</span><span class="p">)</span>
 <span class="p">}</span>
</code></pre></div>    </div>

    <p>在初始 UI 设置正确的情况下，这倒没什么问题。但是如果 UI 状态原本存在不对的话，就将导致接下来的 UI 都是错误的。从更高层次来看，也就是 <code class="highlighter-rouge">state</code> 属性对 UI 的控制不仅仅涉及到新的状态，同时也取决于原有的 <code class="highlighter-rouge">state</code> 值。这会导致一些额外复杂度，是我们想要避免的。理想状态下，UI 的更新应该只和输入有关，而与当前状态无关 (也就是“纯函数式”，我们稍后再具体介绍)。</p>
  </li>
  <li>
    <p><code class="highlighter-rouge">State</code> 难以扩展 - 现在 <code class="highlighter-rouge">State</code> 中只有两个变量 <code class="highlighter-rouge">todos</code> 和 <code class="highlighter-rouge">text</code>，如果 View Controller 中还需要其他的变量，我们可以将它继续添加到 <code class="highlighter-rouge">State</code> 结构体中。不过在实践中这会十分困难，因为我们需要更新所有的 <code class="highlighter-rouge">state</code> 赋值的部分。比如，如果我们添加一个 <code class="highlighter-rouge">loading</code> 来表示正在加载待办：</p>

    <div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code> <span class="kd">struct</span> <span class="kt">State</span> <span class="p">{</span>
     <span class="k">let</span> <span class="nv">todos</span><span class="p">:</span> <span class="p">[</span><span class="kt">String</span><span class="p">]</span>
     <span class="k">let</span> <span class="nv">text</span><span class="p">:</span> <span class="kt">String</span>
     <span class="k">let</span> <span class="nv">loading</span><span class="p">:</span> <span class="kt">Bool</span>
 <span class="p">}</span>
    
 <span class="k">override</span> <span class="kd">func</span> <span class="nf">viewDidLoad</span><span class="p">()</span> <span class="p">{</span>
     <span class="k">super</span><span class="o">.</span><span class="nf">viewDidLoad</span><span class="p">()</span>
        
     <span class="n">state</span> <span class="o">=</span> <span class="kt">State</span><span class="p">(</span><span class="nv">todos</span><span class="p">:</span> <span class="k">self</span><span class="o">.</span><span class="n">state</span><span class="o">.</span><span class="n">todos</span> <span class="o">+</span> <span class="n">data</span><span class="p">,</span> <span class="nv">text</span><span class="p">:</span> <span class="k">self</span><span class="o">.</span><span class="n">state</span><span class="o">.</span><span class="n">text</span><span class="p">,</span> <span class="nv">loading</span><span class="p">:</span> <span class="kc">true</span><span class="p">)</span>
     <span class="kt">ToDoStore</span><span class="o">.</span><span class="n">shared</span><span class="o">.</span><span class="n">getToDoItems</span> <span class="p">{</span> <span class="p">(</span><span class="n">data</span><span class="p">)</span> <span class="k">in</span>
         <span class="k">self</span><span class="o">.</span><span class="n">state</span> <span class="o">=</span> <span class="kt">State</span><span class="p">(</span><span class="nv">todos</span><span class="p">:</span> <span class="k">self</span><span class="o">.</span><span class="n">state</span><span class="o">.</span><span class="n">todos</span> <span class="o">+</span> <span class="n">data</span><span class="p">,</span> <span class="nv">text</span><span class="p">:</span> <span class="k">self</span><span class="o">.</span><span class="n">state</span><span class="o">.</span><span class="n">text</span><span class="p">,</span> <span class="nv">loading</span><span class="p">:</span> <span class="kc">false</span><span class="p">)</span>
     <span class="p">}</span>
 <span class="p">}</span>
</code></pre></div>    </div>

    <p>除此之外，像是添加待办，删除待办等存在 <code class="highlighter-rouge">state</code> 赋值的地方，我们都需要在原来的初始化方法上加上 <code class="highlighter-rouge">loading</code> 参数。试想，如果我们稍后又添加了一个变量，我们则需要再次维护所有这些地方，这显然是无法接受的。</p>

    <p>当然，因为 <code class="highlighter-rouge">State</code> 是值类型，我们可以将 <code class="highlighter-rouge">State</code> 中的变量声明从 <code class="highlighter-rouge">let</code> 改为 <code class="highlighter-rouge">var</code>，这样我们就可以直接设置 <code class="highlighter-rouge">state</code> 中的属性了，例如：</p>

    <div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code> <span class="n">state</span><span class="o">.</span><span class="n">todos</span> <span class="o">=</span> <span class="n">state</span><span class="o">.</span><span class="n">todos</span> <span class="o">+</span> <span class="n">data</span>
 <span class="n">state</span><span class="o">.</span><span class="n">loading</span> <span class="o">=</span> <span class="kc">true</span>
</code></pre></div>    </div>

    <p>这种情况下，<code class="highlighter-rouge">State</code> 的 <code class="highlighter-rouge">didSet</code> 将被调用多次，虽然不太舒服，但倒也不是很大的问题。更关键的地方在于，这样一来我们又将状态的维护零散地分落在各个地方。当状态中的变量越来越多，而且状态自身之间有所依赖的话，这么做又将我们置于麻烦之中。我们还需要注意，如果 <code class="highlighter-rouge">State</code> 中包含引用类型，那么它将失去完全的值语义，也就是说，如果你去改变了 <code class="highlighter-rouge">state</code> 中引用类型里的某个变量时，<code class="highlighter-rouge">state</code> 的 <code class="highlighter-rouge">didSet</code> 将不会被调用。这让我们在使用时需要如履薄冰，一旦这种情况发生，调试也会相对困难。</p>
  </li>
  <li>Data Source 重用 - 我们其实有机会将 Table View 的 Data Source 部分提取出来，让它在不同的 View Controller 中被重复利用。但是现在新引入的 <code class="highlighter-rouge">state</code> 阻止了这一可能性。如果我们想要重用 <code class="highlighter-rouge">dataSource</code>，我们需要将 <code class="highlighter-rouge">state.todos</code> 从中分离出来，或者是找一种方法在 <code class="highlighter-rouge">dataSource</code> 中同步待办事项的 model。</li>
  <li>异步操作的测试 - 在 <code class="highlighter-rouge">TableViewController</code> 的测试中，有一个地方我们没有覆盖到，那就是 <code class="highlighter-rouge">viewDidLoad</code> 中用来加载待办的 <code class="highlighter-rouge">ToDoStore.shared.getToDoItems</code>。在不引入 stub 的情况下，测试这类异步操作会非常困难，但是引入 stub 本身现在看来也不是特别方便。我们有没有什么好方法可以测试 View Controller 中的异步操作呢？</li>
</ol>

<p>我们可以引入一些改变，来将 <code class="highlighter-rouge">TableViewController</code> 的 UI 部分变为纯函数式实现，并利用单向数据流来驱动 View Controller，就可以解决这些问题。</p>

<h2 id="对-view-controller-的进一步改造">对 View Controller 的进一步改造</h2>

<p>在着手大幅调整代码之前，我想先介绍一些基本概念。</p>

<h3 id="什么是纯函数">什么是纯函数</h3>

<p>纯函数 (Pure Function) 是指一个函数如果有相同的输入，则它产生相同的输出。换言之，也就是一个函数的动作不依赖于外部变量之类的状态，一旦输入给定，那么输出则唯一确定。对于 app 而言，我们总是会和一定的用户输入打交道，也必然会需要按照用户的输入和已知状态来更新 UI 作为“输出”。所以在 app 中，特别是 View Controller 中操作 UI 的部分，我会倾向于将“纯函数”定义为：在确定的输入下，某个函数给出确定的 UI。</p>

<p>上面的 <code class="highlighter-rouge">State</code> 为我们打造一个纯函数的 View Controller 提供了坚实的一步，但是它还并不是纯函数。对于任意的新的 <code class="highlighter-rouge">state</code>，输出的 UI 在一定程度上还是依赖于原来的 <code class="highlighter-rouge">state</code>。不过我们可以通过将原来的 <code class="highlighter-rouge">state</code> 提取出来，换成一个用于更新 UI 的纯函数，即可解决这个问题。新的函数签名看起来大概会是这样：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">func</span> <span class="nf">updateViews</span><span class="p">(</span><span class="nv">state</span><span class="p">:</span> <span class="kt">State</span><span class="p">,</span> <span class="nv">previousState</span><span class="p">:</span> <span class="kt">State</span><span class="p">?)</span>
</code></pre></div></div>

<p>这样，当我们给定原状态和现状态时，将得到确定的 UI，我们稍后会来看看这个方法的具体实现。</p>

<h3 id="单向数据流">单向数据流</h3>

<p>我们想要对 State View Controller 做的另一个改进是简化和统一状态维护的相关工作。我们知道，任何新的状态都是在原有状态的基础上通过一些改变所得到的。举例来说，在待办事项的 demo 中，新加一个待办意味着在原状态的 <code class="highlighter-rouge">state.todos</code> 的基础上，接收到用户的添加的行为，然后在数组中加上待办事项，并输出新的状态：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">if</span> <span class="n">userWantToAddItem</span> <span class="p">{</span>
    <span class="n">state</span><span class="o">.</span><span class="n">todos</span> <span class="o">=</span> <span class="n">state</span><span class="o">.</span><span class="n">todos</span> <span class="o">+</span> <span class="p">[</span><span class="n">item</span><span class="p">]</span>
<span class="p">}</span>
</code></pre></div></div>

<p>其他的操作也皆是如此。将这个过成进行一些抽象，我们可以得到这样一个公式：</p>

<div class="highlighter-rouge"><div class="highlight"><pre class="highlight"><code>新状态 = f(旧状态, 用户行为)
</code></pre></div></div>

<p>或者用 Swift 的语言，就是：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">func</span> <span class="nf">reducer</span><span class="p">(</span><span class="nv">state</span><span class="p">:</span> <span class="kt">State</span><span class="p">,</span> <span class="nv">userAction</span><span class="p">:</span> <span class="kt">Action</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="kt">State</span>
</code></pre></div></div>

<p>如果你对函数式编程有所了解，应该很容易看出，这其实就是 <code class="highlighter-rouge">reduce</code> 函数的 <code class="highlighter-rouge">transformer</code>，它接受一个已有状态 <code class="highlighter-rouge">State</code> 和一个输入 <code class="highlighter-rouge">Action</code>，将 <code class="highlighter-rouge">Action</code> 作用于 <code class="highlighter-rouge">state</code>，并给出新的 <code class="highlighter-rouge">State</code>。结合 Swift 标准库中的 <code class="highlighter-rouge">reduce</code> 的函数签名，我们可以轻而易举地看到两者的关联：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">func</span> <span class="n">reduce</span><span class="o">&lt;</span><span class="kt">Result</span><span class="o">&gt;</span><span class="p">(</span><span class="n">_</span> <span class="nv">initialResult</span><span class="p">:</span> <span class="kt">Result</span><span class="p">,</span> 
                    <span class="n">_</span> <span class="nv">nextPartialResult</span><span class="p">:</span> <span class="p">(</span><span class="kt">Result</span><span class="p">,</span> <span class="kt">Element</span><span class="p">)</span> <span class="k">throws</span> <span class="o">-&gt;</span> <span class="kt">Result</span><span class="p">)</span> <span class="k">rethrows</span> <span class="o">-&gt;</span> <span class="kt">Result</span>
</code></pre></div></div>

<p>其中 <code class="highlighter-rouge">reducer</code> 对应的正是 <code class="highlighter-rouge">reduce</code> 中的 <code class="highlighter-rouge">nextPartialResult</code> 部分，这也是我们将它称为 <code class="highlighter-rouge">reducer</code> 的原因。</p>

<p>有了 <code class="highlighter-rouge">reducer(state: State, userAction: Action) -&gt; State</code>，接下来我们就可以将用户操作抽象为 <code class="highlighter-rouge">Action</code>，并将所有的状态更新集中处理了。为了让这个过程一般化，我们会统一使用一个 <code class="highlighter-rouge">Store</code> 类型来存储状态，并通过向 <code class="highlighter-rouge">Store</code> 发送 <code class="highlighter-rouge">Action</code> 来更新其中的状态。而希望接收到状态更新的对象 (这个例子中是 <code class="highlighter-rouge">TableViewController</code> 实例) 可以订阅状态变化，以更新 UI。订阅者不参与直接改变状态，而只是发送可能改变状态的行为，然后接受状态变化并更新 UI，以此形成单向的数据流动。而因为更新 UI 的代码将会是纯函数的，所以 View Controller 的 UI 也将是可预期及可测试的。</p>

<h3 id="异步状态">异步状态</h3>

<p>对于像 <code class="highlighter-rouge">ToDoStore.shared.getToDoItems</code> 这样的异步操作，我们也希望能够纳入到 <code class="highlighter-rouge">Action</code> 和 <code class="highlighter-rouge">reducer</code> 的体系中。异步操作对于状态的立即改变 (比如设置 <code class="highlighter-rouge">state.loading</code> 并显示一个 Loading Indicator)，我们可以通过向 <code class="highlighter-rouge">State</code> 中添加成员来达到。要触发这个异步操作，我们可以为它添加一个新的 <code class="highlighter-rouge">Action</code>，相对于普通 <code class="highlighter-rouge">Action</code> 仅仅只是改变 <code class="highlighter-rouge">state</code>，我们希望它还能有一定“副作用”，也就是在订阅者中能实际触发这个异步操作。这需要我们稍微更新一下 <code class="highlighter-rouge">reducer</code> 的定义，除了返回新的 <code class="highlighter-rouge">State</code> 以外，我们还希望对异步操作返回一个额外的 <code class="highlighter-rouge">Command</code>：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">func</span> <span class="nf">reducer</span><span class="p">(</span><span class="nv">state</span><span class="p">:</span> <span class="kt">State</span><span class="p">,</span> <span class="nv">userAction</span><span class="p">:</span> <span class="kt">Action</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="p">(</span><span class="kt">State</span><span class="p">,</span> <span class="kt">Command</span><span class="p">?)</span>
</code></pre></div></div>

<p><code class="highlighter-rouge">Command</code> 只是触发异步操作的手段，它不应该和状态变化有关，所以它没有出现在 <code class="highlighter-rouge">reducer</code> 的输入一侧。如果你现在不太理解的话也没有关系，先只需要记住这个函数签名，我们会在之后的例子中详细地看到这部分的工作方式。</p>

<p>将这些结合起来，我们将要实现的 View Controller 的架构类似于下图：</p>

<p><img src="/assets/images/2017/view-controller-states.svg" alt=""></p>

<h3 id="使用单向数据流和-reducer-改进-view-controller">使用单向数据流和 reducer 改进 View Controller</h3>

<p>准备工作够多了，让我们来在 State View Controller 的基础上进行改进吧。</p>

<p>为了能够尽量通用，我们先来定义几个协议：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">protocol</span> <span class="kt">ActionType</span> <span class="p">{}</span>
<span class="kd">protocol</span> <span class="kt">StateType</span> <span class="p">{}</span>
<span class="kd">protocol</span> <span class="kt">CommandType</span> <span class="p">{}</span>
</code></pre></div></div>

<p>除了限制协议类型以外，上面这几个 <code class="highlighter-rouge">protocol</code> 并没有其他特别的意义。接下来，我们在 <code class="highlighter-rouge">TableViewController</code> 中定义对应的 <code class="highlighter-rouge">Action</code>，<code class="highlighter-rouge">State</code> 和 <code class="highlighter-rouge">Command</code>：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">class</span> <span class="kt">TableViewController</span><span class="p">:</span> <span class="kt">UITableViewController</span> <span class="p">{</span>
    
    <span class="kd">struct</span> <span class="kt">State</span><span class="p">:</span> <span class="kt">StateType</span> <span class="p">{</span>
        <span class="k">var</span> <span class="nv">dataSource</span> <span class="o">=</span> <span class="kt">TableViewControllerDataSource</span><span class="p">(</span><span class="nv">todos</span><span class="p">:</span> <span class="p">[],</span> <span class="nv">owner</span><span class="p">:</span> <span class="kc">nil</span><span class="p">)</span>
        <span class="k">var</span> <span class="nv">text</span><span class="p">:</span> <span class="kt">String</span> <span class="o">=</span> <span class="s">""</span>
    <span class="p">}</span>
    
    <span class="kd">enum</span> <span class="kt">Action</span><span class="p">:</span> <span class="kt">ActionType</span> <span class="p">{</span>
        <span class="k">case</span> <span class="nf">updateText</span><span class="p">(</span><span class="nv">text</span><span class="p">:</span> <span class="kt">String</span><span class="p">)</span>
        <span class="k">case</span> <span class="nf">addToDos</span><span class="p">(</span><span class="nv">items</span><span class="p">:</span> <span class="p">[</span><span class="kt">String</span><span class="p">])</span>
        <span class="k">case</span> <span class="nf">removeToDo</span><span class="p">(</span><span class="nv">index</span><span class="p">:</span> <span class="kt">Int</span><span class="p">)</span>
        <span class="k">case</span> <span class="n">loadToDos</span>
    <span class="p">}</span>
    
    <span class="kd">enum</span> <span class="kt">Command</span><span class="p">:</span> <span class="kt">CommandType</span> <span class="p">{</span>
        <span class="k">case</span> <span class="nf">loadToDos</span><span class="p">(</span><span class="nv">completion</span><span class="p">:</span> <span class="p">([</span><span class="kt">String</span><span class="p">])</span> <span class="o">-&gt;</span> <span class="kt">Void</span> <span class="p">)</span>
    <span class="p">}</span>
    
    
    <span class="c1">//...</span>
<span class="p">}</span>
</code></pre></div></div>

<p>为了将 <code class="highlighter-rouge">dataSource</code> 提取出来，我们在 <code class="highlighter-rouge">State</code> 中把原来的 <code class="highlighter-rouge">todos</code> 换成了整个的 <code class="highlighter-rouge">dataSource</code>。<code class="highlighter-rouge">TableViewControllerDataSource</code> 就是标准的 <code class="highlighter-rouge">UITableViewDataSource</code>，它包含 <code class="highlighter-rouge">todos</code> 和用来作为 <code class="highlighter-rouge">inputCell</code> 设定 <code class="highlighter-rouge">delegate</code> 的 <code class="highlighter-rouge">owner</code>。基本上就是将原来 <code class="highlighter-rouge">TableViewController</code> 的 Data Source 部分的代码搬过去，部分关键代码如下：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">class</span> <span class="kt">TableViewControllerDataSource</span><span class="p">:</span> <span class="kt">NSObject</span><span class="p">,</span> <span class="kt">UITableViewDataSource</span> <span class="p">{</span>

    <span class="k">var</span> <span class="nv">todos</span><span class="p">:</span> <span class="p">[</span><span class="kt">String</span><span class="p">]</span>
    <span class="k">weak</span> <span class="k">var</span> <span class="nv">owner</span><span class="p">:</span> <span class="kt">TableViewController</span><span class="p">?</span>
    
    <span class="nf">init</span><span class="p">(</span><span class="nv">todos</span><span class="p">:</span> <span class="p">[</span><span class="kt">String</span><span class="p">],</span> <span class="nv">owner</span><span class="p">:</span> <span class="kt">TableViewController</span><span class="p">?)</span> <span class="p">{</span>
        <span class="k">self</span><span class="o">.</span><span class="n">todos</span> <span class="o">=</span> <span class="n">todos</span>
        <span class="k">self</span><span class="o">.</span><span class="n">owner</span> <span class="o">=</span> <span class="n">owner</span>
    <span class="p">}</span>
    
    <span class="c1">//...</span>
    <span class="kd">func</span> <span class="nf">tableView</span><span class="p">(</span><span class="n">_</span> <span class="nv">tableView</span><span class="p">:</span> <span class="kt">UITableView</span><span class="p">,</span> <span class="n">cellForRowAt</span> <span class="nv">indexPath</span><span class="p">:</span> <span class="kt">IndexPath</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="kt">UITableViewCell</span> <span class="p">{</span>
        <span class="c1">//...</span>
            <span class="k">let</span> <span class="nv">cell</span> <span class="o">=</span> <span class="n">tableView</span><span class="o">.</span><span class="nf">dequeueReusableCell</span><span class="p">(</span><span class="nv">withIdentifier</span><span class="p">:</span> <span class="n">inputCellReuseId</span><span class="p">,</span> <span class="nv">for</span><span class="p">:</span> <span class="n">indexPath</span><span class="p">)</span> <span class="k">as!</span> <span class="kt">TableViewInputCell</span>
            <span class="n">cell</span><span class="o">.</span><span class="n">delegate</span> <span class="o">=</span> <span class="n">owner</span>
            <span class="k">return</span> <span class="n">cell</span>
    <span class="p">}</span>

<span class="p">}</span>
</code></pre></div></div>

<p>这是基本的将 Data Source 分离出 View Controller 的方法，本身很简单，也不是本文的重点。</p>

<p>注意 <code class="highlighter-rouge">Command</code> 中包含的 <code class="highlighter-rouge">loadToDos</code> 成员，它关联了一个方法作为结束时的回调，我们稍后会在这个方法里向 <code class="highlighter-rouge">store</code> 发送 <code class="highlighter-rouge">.addToDos</code> 的 <code class="highlighter-rouge">Action</code>。</p>

<p>准备好必要的类型后，我们就可以实现核心的 <code class="highlighter-rouge">reducer</code> 了：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">lazy</span> <span class="k">var</span> <span class="nv">reducer</span><span class="p">:</span> <span class="p">(</span><span class="kt">State</span><span class="p">,</span> <span class="kt">Action</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="p">(</span><span class="nv">state</span><span class="p">:</span> <span class="kt">State</span><span class="p">,</span> <span class="nv">command</span><span class="p">:</span> <span class="kt">Command</span><span class="p">?)</span> <span class="o">=</span> <span class="p">{</span>
    <span class="p">[</span><span class="k">weak</span> <span class="k">self</span><span class="p">]</span> <span class="p">(</span><span class="nv">state</span><span class="p">:</span> <span class="kt">State</span><span class="p">,</span> <span class="nv">action</span><span class="p">:</span> <span class="kt">Action</span><span class="p">)</span> <span class="k">in</span>
    
    <span class="k">var</span> <span class="nv">state</span> <span class="o">=</span> <span class="n">state</span>
    <span class="k">var</span> <span class="nv">command</span><span class="p">:</span> <span class="kt">Command</span><span class="p">?</span> <span class="o">=</span> <span class="kc">nil</span>

    <span class="k">switch</span> <span class="n">action</span> <span class="p">{</span>
    <span class="k">case</span> <span class="o">.</span><span class="nf">updateText</span><span class="p">(</span><span class="k">let</span> <span class="nv">text</span><span class="p">):</span>
        <span class="n">state</span><span class="o">.</span><span class="n">text</span> <span class="o">=</span> <span class="n">text</span>
    <span class="k">case</span> <span class="o">.</span><span class="nf">addToDos</span><span class="p">(</span><span class="k">let</span> <span class="nv">items</span><span class="p">):</span>
        <span class="n">state</span><span class="o">.</span><span class="n">dataSource</span> <span class="o">=</span> <span class="kt">TableViewControllerDataSource</span><span class="p">(</span><span class="nv">todos</span><span class="p">:</span> <span class="n">items</span> <span class="o">+</span> <span class="n">state</span><span class="o">.</span><span class="n">dataSource</span><span class="o">.</span><span class="n">todos</span><span class="p">,</span> <span class="nv">owner</span><span class="p">:</span> <span class="n">state</span><span class="o">.</span><span class="n">dataSource</span><span class="o">.</span><span class="n">owner</span><span class="p">)</span>
    <span class="k">case</span> <span class="o">.</span><span class="nf">removeToDo</span><span class="p">(</span><span class="k">let</span> <span class="nv">index</span><span class="p">):</span>
        <span class="k">let</span> <span class="nv">oldTodos</span> <span class="o">=</span> <span class="n">state</span><span class="o">.</span><span class="n">dataSource</span><span class="o">.</span><span class="n">todos</span>
        <span class="n">state</span><span class="o">.</span><span class="n">dataSource</span> <span class="o">=</span> <span class="kt">TableViewControllerDataSource</span><span class="p">(</span><span class="nv">todos</span><span class="p">:</span> <span class="kt">Array</span><span class="p">(</span><span class="n">oldTodos</span><span class="p">[</span><span class="o">..&lt;</span><span class="n">index</span><span class="p">]</span> <span class="o">+</span> <span class="n">oldTodos</span><span class="p">[(</span><span class="n">index</span> <span class="o">+</span> <span class="mi">1</span><span class="p">)</span><span class="o">...</span><span class="p">]),</span> <span class="nv">owner</span><span class="p">:</span> <span class="n">state</span><span class="o">.</span><span class="n">dataSource</span><span class="o">.</span><span class="n">owner</span><span class="p">)</span>
    <span class="k">case</span> <span class="o">.</span><span class="nv">loadToDos</span><span class="p">:</span>
        <span class="n">command</span> <span class="o">=</span> <span class="kt">Command</span><span class="o">.</span><span class="n">loadToDos</span> <span class="p">{</span> <span class="n">data</span> <span class="k">in</span>
            <span class="c1">// 发送额外的 .addToDos</span>
        <span class="p">}</span>
    <span class="p">}</span>
    <span class="nf">return</span> <span class="p">(</span><span class="n">state</span><span class="p">,</span> <span class="n">command</span><span class="p">)</span>
<span class="p">}</span>
</code></pre></div></div>

<blockquote>
  <p>为了避免 <code class="highlighter-rouge">reducer</code> 持有 <code class="highlighter-rouge">self</code> 而造成内存泄漏，这里我们所实现的是一个 lazy 的 <code class="highlighter-rouge">reducer</code> 成员。其中 <code class="highlighter-rouge">self</code> 被标记为弱引用，这样一来，我们就不需要担心 <code class="highlighter-rouge">store</code>，View Controller 和 <code class="highlighter-rouge">reducer</code> 之间的引用环了。</p>
</blockquote>

<p>对于 <code class="highlighter-rouge">.updateText</code>，<code class="highlighter-rouge">.addToDos</code> 和 <code class="highlighter-rouge">.removeToDo</code>，我们都只是根据已有状态衍生出新的状态。唯一值得注意的是 <code class="highlighter-rouge">.loadToDos</code>，它将让 <code class="highlighter-rouge">reducer</code> 函数返回非空的 <code class="highlighter-rouge">Command</code>。</p>

<p>接下来我们需要一个存储状态和响应 <code class="highlighter-rouge">Action</code> 的类型，我们将它叫做 <code class="highlighter-rouge">Store</code>：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">class</span> <span class="kt">Store</span><span class="o">&lt;</span><span class="kt">A</span><span class="p">:</span> <span class="kt">ActionType</span><span class="p">,</span> <span class="kt">S</span><span class="p">:</span> <span class="kt">StateType</span><span class="p">,</span> <span class="kt">C</span><span class="p">:</span> <span class="kt">CommandType</span><span class="o">&gt;</span> <span class="p">{</span>
    <span class="k">let</span> <span class="nv">reducer</span><span class="p">:</span> <span class="p">(</span><span class="n">_</span> <span class="nv">state</span><span class="p">:</span> <span class="kt">S</span><span class="p">,</span> <span class="n">_</span> <span class="nv">action</span><span class="p">:</span> <span class="kt">A</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="p">(</span><span class="kt">S</span><span class="p">,</span> <span class="kt">C</span><span class="p">?)</span>
    <span class="k">var</span> <span class="nv">subscriber</span><span class="p">:</span> <span class="p">((</span><span class="n">_</span> <span class="nv">state</span><span class="p">:</span> <span class="kt">S</span><span class="p">,</span> <span class="n">_</span> <span class="nv">previousState</span><span class="p">:</span> <span class="kt">S</span><span class="p">,</span> <span class="n">_</span> <span class="nv">command</span><span class="p">:</span> <span class="kt">C</span><span class="p">?)</span> <span class="o">-&gt;</span> <span class="kt">Void</span><span class="p">)?</span>
    <span class="k">var</span> <span class="nv">state</span><span class="p">:</span> <span class="kt">S</span>
    
    <span class="nf">init</span><span class="p">(</span><span class="nv">reducer</span><span class="p">:</span> <span class="kd">@escaping</span> <span class="p">(</span><span class="kt">S</span><span class="p">,</span> <span class="kt">A</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="p">(</span><span class="kt">S</span><span class="p">,</span> <span class="kt">C</span><span class="p">?),</span> <span class="nv">initialState</span><span class="p">:</span> <span class="kt">S</span><span class="p">)</span> <span class="p">{</span>
        <span class="k">self</span><span class="o">.</span><span class="n">reducer</span> <span class="o">=</span> <span class="n">reducer</span>
        <span class="k">self</span><span class="o">.</span><span class="n">state</span> <span class="o">=</span> <span class="n">initialState</span>
    <span class="p">}</span>
    
    <span class="kd">func</span> <span class="nf">dispatch</span><span class="p">(</span><span class="n">_</span> <span class="nv">action</span><span class="p">:</span> <span class="kt">A</span><span class="p">)</span> <span class="p">{</span>
        <span class="k">let</span> <span class="nv">previousState</span> <span class="o">=</span> <span class="n">state</span>
        <span class="k">let</span> <span class="p">(</span><span class="nv">nextState</span><span class="p">,</span> <span class="nv">command</span><span class="p">)</span> <span class="o">=</span> <span class="nf">reducer</span><span class="p">(</span><span class="n">state</span><span class="p">,</span> <span class="n">action</span><span class="p">)</span>
        <span class="n">state</span> <span class="o">=</span> <span class="n">nextState</span>
        <span class="nf">subscriber</span><span class="p">?(</span><span class="n">state</span><span class="p">,</span> <span class="n">previousState</span><span class="p">,</span> <span class="n">command</span><span class="p">)</span>
    <span class="p">}</span>
    
    <span class="kd">func</span> <span class="nf">subscribe</span><span class="p">(</span><span class="n">_</span> <span class="nv">handler</span><span class="p">:</span> <span class="kd">@escaping</span> <span class="p">(</span><span class="kt">S</span><span class="p">,</span> <span class="kt">S</span><span class="p">,</span> <span class="kt">C</span><span class="p">?)</span> <span class="o">-&gt;</span> <span class="kt">Void</span><span class="p">)</span> <span class="p">{</span>
        <span class="k">self</span><span class="o">.</span><span class="n">subscriber</span> <span class="o">=</span> <span class="n">handler</span>
    <span class="p">}</span>
    
    <span class="kd">func</span> <span class="nf">unsubscribe</span><span class="p">()</span> <span class="p">{</span>
        <span class="k">self</span><span class="o">.</span><span class="n">subscriber</span> <span class="o">=</span> <span class="kc">nil</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

<p>千万不要被这些泛型吓到，它们都非常简单。这个 <code class="highlighter-rouge">Store</code> 接受一个 <code class="highlighter-rouge">reducer</code> 和一个初始状态 <code class="highlighter-rouge">initialState</code> 作为输入。它提供了 <code class="highlighter-rouge">dispatch</code> 方法，持有该 <code class="highlighter-rouge">store</code> 的类型可以通过 <code class="highlighter-rouge">dispatch</code> 向其发送 <code class="highlighter-rouge">Action</code>，<code class="highlighter-rouge">store</code> 将根据 <code class="highlighter-rouge">reducer</code> 提供的方式生成新的 <code class="highlighter-rouge">state</code> 和必要的 <code class="highlighter-rouge">command</code>，然后通知它的订阅者。</p>

<p>在 <code class="highlighter-rouge">TableViewController</code> 中增加一个 <code class="highlighter-rouge">store</code> 变量，并在 <code class="highlighter-rouge">viewDidLoad</code> 中初始化它：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">class</span> <span class="kt">TableViewController</span><span class="p">:</span> <span class="kt">UITableViewController</span> <span class="p">{</span>
    <span class="k">var</span> <span class="nv">store</span><span class="p">:</span> <span class="kt">Store</span><span class="o">&lt;</span><span class="kt">Action</span><span class="p">,</span> <span class="kt">State</span><span class="p">,</span> <span class="kt">Command</span><span class="o">&gt;!</span>
    
    <span class="k">override</span> <span class="kd">func</span> <span class="nf">viewDidLoad</span><span class="p">()</span> <span class="p">{</span>
        <span class="k">super</span><span class="o">.</span><span class="nf">viewDidLoad</span><span class="p">()</span>
        
        <span class="k">let</span> <span class="nv">dataSource</span> <span class="o">=</span> <span class="kt">TableViewControllerDataSource</span><span class="p">(</span><span class="nv">todos</span><span class="p">:</span> <span class="p">[],</span> <span class="nv">owner</span><span class="p">:</span> <span class="k">self</span><span class="p">)</span>
        <span class="n">store</span> <span class="o">=</span> <span class="kt">Store</span><span class="o">&lt;</span><span class="kt">Action</span><span class="p">,</span> <span class="kt">State</span><span class="p">,</span> <span class="kt">Command</span><span class="o">&gt;</span><span class="p">(</span><span class="nv">reducer</span><span class="p">:</span> <span class="n">reducer</span><span class="p">,</span> <span class="nv">initialState</span><span class="p">:</span> <span class="kt">State</span><span class="p">(</span><span class="nv">dataSource</span><span class="p">:</span> <span class="n">dataSource</span><span class="p">,</span> <span class="nv">text</span><span class="p">:</span> <span class="s">""</span><span class="p">))</span>
        
        <span class="c1">// 订阅 store</span>
        <span class="n">store</span><span class="o">.</span><span class="n">subscribe</span> <span class="p">{</span> <span class="p">[</span><span class="k">weak</span> <span class="k">self</span><span class="p">]</span> <span class="n">state</span><span class="p">,</span> <span class="n">previousState</span><span class="p">,</span> <span class="n">command</span> <span class="k">in</span>
            <span class="k">self</span><span class="p">?</span><span class="o">.</span><span class="nf">stateDidChanged</span><span class="p">(</span><span class="nv">state</span><span class="p">:</span> <span class="n">state</span><span class="p">,</span> <span class="nv">previousState</span><span class="p">:</span> <span class="n">previousState</span><span class="p">,</span> <span class="nv">command</span><span class="p">:</span> <span class="n">command</span><span class="p">)</span>
        <span class="p">}</span>
        
        <span class="c1">// 初始化 UI</span>
        <span class="nf">stateDidChanged</span><span class="p">(</span><span class="nv">state</span><span class="p">:</span> <span class="n">store</span><span class="o">.</span><span class="n">state</span><span class="p">,</span> <span class="nv">previousState</span><span class="p">:</span> <span class="kc">nil</span><span class="p">,</span> <span class="nv">command</span><span class="p">:</span> <span class="kc">nil</span><span class="p">)</span>
        
        <span class="c1">// 开始异步加载 ToDos</span>
        <span class="n">store</span><span class="o">.</span><span class="nf">dispatch</span><span class="p">(</span><span class="o">.</span><span class="n">loadToDos</span><span class="p">)</span>
    <span class="p">}</span>
    
    <span class="c1">//...</span>
<span class="p">}</span>
</code></pre></div></div>

<p>将 <code class="highlighter-rouge">stateDidChanged</code> 添加到 <code class="highlighter-rouge">store.subscribe</code> 后，每次 <code class="highlighter-rouge">store</code> 状态改变时，<code class="highlighter-rouge">stateDidChanged</code> 都将被调用。现在我们还没有实现这个方法，它的具体内容如下：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code>    <span class="kd">func</span> <span class="nf">stateDidChanged</span><span class="p">(</span><span class="nv">state</span><span class="p">:</span> <span class="kt">State</span><span class="p">,</span> <span class="nv">previousState</span><span class="p">:</span> <span class="kt">State</span><span class="p">?,</span> <span class="nv">command</span><span class="p">:</span> <span class="kt">Command</span><span class="p">?)</span> <span class="p">{</span>
        
        <span class="k">if</span> <span class="k">let</span> <span class="nv">command</span> <span class="o">=</span> <span class="n">command</span> <span class="p">{</span>
            <span class="k">switch</span> <span class="n">command</span> <span class="p">{</span>
            <span class="k">case</span> <span class="o">.</span><span class="nf">loadToDos</span><span class="p">(</span><span class="k">let</span> <span class="nv">handler</span><span class="p">):</span>
                <span class="kt">ToDoStore</span><span class="o">.</span><span class="n">shared</span><span class="o">.</span><span class="nf">getToDoItems</span><span class="p">(</span><span class="nv">completionHandler</span><span class="p">:</span> <span class="n">handler</span><span class="p">)</span>
            <span class="p">}</span>
        <span class="p">}</span>
        
        <span class="k">if</span> <span class="n">previousState</span> <span class="o">==</span> <span class="kc">nil</span> <span class="o">||</span> <span class="n">previousState</span><span class="o">!.</span><span class="n">dataSource</span><span class="o">.</span><span class="n">todos</span> <span class="o">!=</span> <span class="n">state</span><span class="o">.</span><span class="n">dataSource</span><span class="o">.</span><span class="n">todos</span> <span class="p">{</span>
            <span class="k">let</span> <span class="nv">dataSource</span> <span class="o">=</span> <span class="n">state</span><span class="o">.</span><span class="n">dataSource</span>
            <span class="n">tableView</span><span class="o">.</span><span class="n">dataSource</span> <span class="o">=</span> <span class="n">dataSource</span>
            <span class="n">tableView</span><span class="o">.</span><span class="nf">reloadData</span><span class="p">()</span>
            <span class="n">title</span> <span class="o">=</span> <span class="s">"TODO - (</span><span class="se">\(</span><span class="n">dataSource</span><span class="o">.</span><span class="n">todos</span><span class="o">.</span><span class="n">count</span><span class="se">)</span><span class="s">)"</span>
        <span class="p">}</span>
        
        <span class="k">if</span> <span class="n">previousState</span> <span class="o">==</span> <span class="kc">nil</span> <span class="o">||</span> <span class="n">previousState</span><span class="o">!.</span><span class="n">text</span> <span class="o">!=</span> <span class="n">state</span><span class="o">.</span><span class="n">text</span> <span class="p">{</span>
            <span class="k">let</span> <span class="nv">isItemLengthEnough</span> <span class="o">=</span> <span class="n">state</span><span class="o">.</span><span class="n">text</span><span class="o">.</span><span class="n">count</span> <span class="o">&gt;=</span> <span class="mi">3</span>
            <span class="n">navigationItem</span><span class="o">.</span><span class="n">rightBarButtonItem</span><span class="p">?</span><span class="o">.</span><span class="n">isEnabled</span> <span class="o">=</span> <span class="n">isItemLengthEnough</span>
            
            <span class="k">let</span> <span class="nv">inputIndexPath</span> <span class="o">=</span> <span class="kt">IndexPath</span><span class="p">(</span><span class="nv">row</span><span class="p">:</span> <span class="mi">0</span><span class="p">,</span> <span class="nv">section</span><span class="p">:</span> <span class="kt">TableViewControllerDataSource</span><span class="o">.</span><span class="kt">Section</span><span class="o">.</span><span class="n">input</span><span class="o">.</span><span class="n">rawValue</span><span class="p">)</span>
            <span class="k">let</span> <span class="nv">inputCell</span> <span class="o">=</span> <span class="n">tableView</span><span class="o">.</span><span class="nf">cellForRow</span><span class="p">(</span><span class="nv">at</span><span class="p">:</span> <span class="n">inputIndexPath</span><span class="p">)</span> <span class="k">as?</span> <span class="kt">TableViewInputCell</span>
            <span class="n">inputCell</span><span class="p">?</span><span class="o">.</span><span class="n">textField</span><span class="o">.</span><span class="n">text</span> <span class="o">=</span> <span class="n">state</span><span class="o">.</span><span class="n">text</span>
        <span class="p">}</span>
    <span class="p">}</span>
</code></pre></div></div>

<p>同时，我们就可以把之前 <code class="highlighter-rouge">Command.loadTodos</code> 的回调补全了：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">lazy</span> <span class="k">var</span> <span class="nv">reducer</span><span class="p">:</span> <span class="p">(</span><span class="kt">State</span><span class="p">,</span> <span class="kt">Action</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="p">(</span><span class="nv">state</span><span class="p">:</span> <span class="kt">State</span><span class="p">,</span> <span class="nv">command</span><span class="p">:</span> <span class="kt">Command</span><span class="p">?)</span> <span class="o">=</span> <span class="p">{</span>
    <span class="p">[</span><span class="k">weak</span> <span class="k">self</span><span class="p">]</span> <span class="p">(</span><span class="nv">state</span><span class="p">:</span> <span class="kt">State</span><span class="p">,</span> <span class="nv">action</span><span class="p">:</span> <span class="kt">Action</span><span class="p">)</span> <span class="k">in</span>
    
    <span class="k">var</span> <span class="nv">state</span> <span class="o">=</span> <span class="n">state</span>
    <span class="k">var</span> <span class="nv">command</span><span class="p">:</span> <span class="kt">Command</span><span class="p">?</span> <span class="o">=</span> <span class="kc">nil</span>

    <span class="k">switch</span> <span class="n">action</span> <span class="p">{</span>
    <span class="c1">// ...</span>
    <span class="k">case</span> <span class="o">.</span><span class="nv">loadToDos</span><span class="p">:</span>
        <span class="n">command</span> <span class="o">=</span> <span class="kt">Command</span><span class="o">.</span><span class="n">loadToDos</span> <span class="p">{</span> <span class="n">data</span> <span class="k">in</span>
            <span class="c1">// 发送额外的 .addToDos</span>
            <span class="k">self</span><span class="p">?</span><span class="o">.</span><span class="n">store</span><span class="o">.</span><span class="nf">dispatch</span><span class="p">(</span><span class="o">.</span><span class="nf">addToDos</span><span class="p">(</span><span class="nv">items</span><span class="p">:</span> <span class="n">data</span><span class="p">))</span>
        <span class="p">}</span>
    <span class="p">}</span>
    <span class="nf">return</span> <span class="p">(</span><span class="n">state</span><span class="p">,</span> <span class="n">command</span><span class="p">)</span>
<span class="p">}</span>
</code></pre></div></div>

<p><code class="highlighter-rouge">stateDidChanged</code> 现在是一个纯函数式的 UI 更新方法，它的输出 (UI) 只取决于输入的 <code class="highlighter-rouge">state</code> 和 <code class="highlighter-rouge">previousState</code>。另一个输入 <code class="highlighter-rouge">Command</code> 负责触发一些不影响输出的“副作用”，在实践中，除了发送请求这样的异步操作外，View Controller 的转换，弹窗之类的交互都可以通过 <code class="highlighter-rouge">Command</code> 来进行。<code class="highlighter-rouge">Command</code> 本身不应该影响 <code class="highlighter-rouge">State</code> 的转换，它需要通过再次发送 <code class="highlighter-rouge">Action</code> 来改变状态，以此才能影响 UI。</p>

<p>到这里，我们基本上拥有所有的部件了。最后的收尾工作相当容易，把之前的直接的状态变更代码换成事件发送即可：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">override</span> <span class="kd">func</span> <span class="nf">tableView</span><span class="p">(</span><span class="n">_</span> <span class="nv">tableView</span><span class="p">:</span> <span class="kt">UITableView</span><span class="p">,</span> <span class="n">didSelectRowAt</span> <span class="nv">indexPath</span><span class="p">:</span> <span class="kt">IndexPath</span><span class="p">)</span> <span class="p">{</span>
    <span class="k">guard</span> <span class="n">indexPath</span><span class="o">.</span><span class="n">section</span> <span class="o">==</span> <span class="kt">TableViewControllerDataSource</span><span class="o">.</span><span class="kt">Section</span><span class="o">.</span><span class="n">todos</span><span class="o">.</span><span class="n">rawValue</span> <span class="k">else</span> <span class="p">{</span> <span class="k">return</span> <span class="p">}</span>
    <span class="n">store</span><span class="o">.</span><span class="nf">dispatch</span><span class="p">(</span><span class="o">.</span><span class="nf">removeToDo</span><span class="p">(</span><span class="nv">index</span><span class="p">:</span> <span class="n">indexPath</span><span class="o">.</span><span class="n">row</span><span class="p">))</span>
<span class="p">}</span>
    
<span class="kd">@IBAction</span> <span class="kd">func</span> <span class="nf">addButtonPressed</span><span class="p">(</span><span class="n">_</span> <span class="nv">sender</span><span class="p">:</span> <span class="kt">Any</span><span class="p">)</span> <span class="p">{</span>
    <span class="n">store</span><span class="o">.</span><span class="nf">dispatch</span><span class="p">(</span><span class="o">.</span><span class="nf">addToDos</span><span class="p">(</span><span class="nv">items</span><span class="p">:</span> <span class="p">[</span><span class="n">store</span><span class="o">.</span><span class="n">state</span><span class="o">.</span><span class="n">text</span><span class="p">]))</span>
    <span class="n">store</span><span class="o">.</span><span class="nf">dispatch</span><span class="p">(</span><span class="o">.</span><span class="nf">updateText</span><span class="p">(</span><span class="nv">text</span><span class="p">:</span> <span class="s">""</span><span class="p">))</span>
<span class="p">}</span>

<span class="kd">func</span> <span class="nf">inputChanged</span><span class="p">(</span><span class="nv">cell</span><span class="p">:</span> <span class="kt">TableViewInputCell</span><span class="p">,</span> <span class="nv">text</span><span class="p">:</span> <span class="kt">String</span><span class="p">)</span> <span class="p">{</span>
    <span class="n">store</span><span class="o">.</span><span class="nf">dispatch</span><span class="p">(</span><span class="o">.</span><span class="nf">updateText</span><span class="p">(</span><span class="nv">text</span><span class="p">:</span> <span class="n">text</span><span class="p">))</span>
<span class="p">}</span>
</code></pre></div></div>

<h3 id="测试纯函数式-view-controller">测试纯函数式 View Controller</h3>

<p>折腾了这么半天，归根结底，其实我们想要的是一个高度可测试的 View Controller。基于高度可测试性，我们就能拥有高度的可维护性。<code class="highlighter-rouge">stateDidChanged</code> 现在是一个纯函数，与 <code class="highlighter-rouge">controller</code> 的当前状态无关，测试它将非常容易：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">func</span> <span class="nf">testUpdateView</span><span class="p">()</span> <span class="p">{</span>
    <span class="k">let</span> <span class="nv">state1</span> <span class="o">=</span> <span class="kt">TableViewController</span><span class="o">.</span><span class="kt">State</span><span class="p">(</span>
        <span class="nv">dataSource</span><span class="p">:</span><span class="kt">TableViewControllerDataSource</span><span class="p">(</span><span class="nv">todos</span><span class="p">:</span> <span class="p">[],</span> <span class="nv">owner</span><span class="p">:</span> <span class="kc">nil</span><span class="p">),</span>
        <span class="nv">text</span><span class="p">:</span> <span class="s">""</span>
    <span class="p">)</span>
    <span class="c1">// 从 nil 状态转换为 state1</span>
    <span class="n">controller</span><span class="o">.</span><span class="nf">stateDidChanged</span><span class="p">(</span><span class="nv">state</span><span class="p">:</span> <span class="n">state1</span><span class="p">,</span> <span class="nv">previousState</span><span class="p">:</span> <span class="kc">nil</span><span class="p">,</span> <span class="nv">command</span><span class="p">:</span> <span class="kc">nil</span><span class="p">)</span>
    <span class="kt">XCTAssertEqual</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">title</span><span class="p">,</span> <span class="s">"TODO - (0)"</span><span class="p">)</span>
    <span class="kt">XCTAssertEqual</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">tableView</span><span class="o">.</span><span class="nf">numberOfRows</span><span class="p">(</span><span class="nv">inSection</span><span class="p">:</span> <span class="kt">TableViewControllerDataSource</span><span class="o">.</span><span class="kt">Section</span><span class="o">.</span><span class="n">todos</span><span class="o">.</span><span class="n">rawValue</span><span class="p">),</span> <span class="mi">0</span><span class="p">)</span>
    <span class="kt">XCTAssertFalse</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">navigationItem</span><span class="o">.</span><span class="n">rightBarButtonItem</span><span class="o">!.</span><span class="n">isEnabled</span><span class="p">)</span>
        
    <span class="k">let</span> <span class="nv">state2</span> <span class="o">=</span> <span class="kt">TableViewController</span><span class="o">.</span><span class="kt">State</span><span class="p">(</span>
        <span class="nv">dataSource</span><span class="p">:</span><span class="kt">TableViewControllerDataSource</span><span class="p">(</span><span class="nv">todos</span><span class="p">:</span> <span class="p">[</span><span class="s">"1"</span><span class="p">,</span> <span class="s">"3"</span><span class="p">],</span> <span class="nv">owner</span><span class="p">:</span> <span class="kc">nil</span><span class="p">),</span>
        <span class="nv">text</span><span class="p">:</span> <span class="s">"Hello"</span>
    <span class="p">)</span>
    <span class="c1">// 从 state1 状态转换为 state2</span>
    <span class="n">controller</span><span class="o">.</span><span class="nf">stateDidChanged</span><span class="p">(</span><span class="nv">state</span><span class="p">:</span> <span class="n">state2</span><span class="p">,</span> <span class="nv">previousState</span><span class="p">:</span> <span class="n">state1</span><span class="p">,</span> <span class="nv">command</span><span class="p">:</span> <span class="kc">nil</span><span class="p">)</span>
    <span class="kt">XCTAssertEqual</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">title</span><span class="p">,</span> <span class="s">"TODO - (2)"</span><span class="p">)</span>
    <span class="kt">XCTAssertEqual</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">tableView</span><span class="o">.</span><span class="nf">numberOfRows</span><span class="p">(</span><span class="nv">inSection</span><span class="p">:</span> <span class="kt">TableViewControllerDataSource</span><span class="o">.</span><span class="kt">Section</span><span class="o">.</span><span class="n">todos</span><span class="o">.</span><span class="n">rawValue</span><span class="p">),</span> <span class="mi">2</span><span class="p">)</span>
    <span class="kt">XCTAssertEqual</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">tableView</span><span class="o">.</span><span class="nf">cellForRow</span><span class="p">(</span><span class="nv">at</span><span class="p">:</span> <span class="nf">todoItemIndexPath</span><span class="p">(</span><span class="nv">row</span><span class="p">:</span> <span class="mi">1</span><span class="p">))?</span><span class="o">.</span><span class="n">textLabel</span><span class="p">?</span><span class="o">.</span><span class="n">text</span><span class="p">,</span> <span class="s">"3"</span><span class="p">)</span>
    <span class="kt">XCTAssertTrue</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">navigationItem</span><span class="o">.</span><span class="n">rightBarButtonItem</span><span class="o">!.</span><span class="n">isEnabled</span><span class="p">)</span>
<span class="p">}</span>
</code></pre></div></div>

<p>作为单元测试，能覆盖产品代码就意味着覆盖了绝大多数使用情况。除此之外，如果你愿意，你也可以写出各种状态间的转换，覆盖尽可能多的边界情况。这可以保证你的代码不会因为新的修改发生退化。</p>

<p>虽然我们没有明说，但是 <code class="highlighter-rouge">TableViewController</code> 中的另一个重要的函数 <code class="highlighter-rouge">reducer</code> 也是纯函数。对它的测试同样简单，比如：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">func</span> <span class="nf">testReducerUpdateTextFromEmpty</span><span class="p">()</span> <span class="p">{</span>
    <span class="k">let</span> <span class="nv">initState</span> <span class="o">=</span> <span class="kt">TableViewController</span><span class="o">.</span><span class="kt">State</span><span class="p">()</span>
    <span class="k">let</span> <span class="nv">state</span> <span class="o">=</span> <span class="n">controller</span><span class="o">.</span><span class="nf">reducer</span><span class="p">(</span><span class="n">initState</span><span class="p">,</span> <span class="o">.</span><span class="nf">updateText</span><span class="p">(</span><span class="nv">text</span><span class="p">:</span> <span class="s">"123"</span><span class="p">))</span><span class="o">.</span><span class="n">state</span>
    <span class="kt">XCTAssertEqual</span><span class="p">(</span><span class="n">state</span><span class="o">.</span><span class="n">text</span><span class="p">,</span> <span class="s">"123"</span><span class="p">)</span>
<span class="p">}</span>
</code></pre></div></div>

<p>输出的 <code class="highlighter-rouge">state</code> 只与输入的 <code class="highlighter-rouge">initState</code> 和 <code class="highlighter-rouge">action</code> 有关，它与 View Controller 的状态完全无关。<code class="highlighter-rouge">reducer</code> 中的其他方法的测试如出一辙，在此不再赘言。</p>

<p>最后，让我们来看看 State View Controller 中没有被测试的加载部分的内容。由于现在加载新的待办事项也是由一个 <code class="highlighter-rouge">Action</code> 来触发的，我们可以通过检查 <code class="highlighter-rouge">reducer</code> 返回的 <code class="highlighter-rouge">Command</code> 来确认加载的结果：</p>

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">func</span> <span class="nf">testLoadToDos</span><span class="p">()</span> <span class="p">{</span>
    <span class="k">let</span> <span class="nv">initState</span> <span class="o">=</span> <span class="kt">TableViewController</span><span class="o">.</span><span class="kt">State</span><span class="p">()</span>
    <span class="k">let</span> <span class="p">(</span><span class="nv">_</span><span class="p">,</span> <span class="nv">command</span><span class="p">)</span> <span class="o">=</span> <span class="n">controller</span><span class="o">.</span><span class="nf">reducer</span><span class="p">(</span><span class="n">initState</span><span class="p">,</span> <span class="o">.</span><span class="n">loadToDos</span><span class="p">)</span>
    <span class="kt">XCTAssertNotNil</span><span class="p">(</span><span class="n">command</span><span class="p">)</span>
    <span class="k">switch</span> <span class="n">command</span><span class="o">!</span> <span class="p">{</span>
    <span class="k">case</span> <span class="o">.</span><span class="nf">loadToDos</span><span class="p">(</span><span class="k">let</span> <span class="nv">handler</span><span class="p">):</span>
        <span class="nf">handler</span><span class="p">([</span><span class="s">"2"</span><span class="p">,</span> <span class="s">"3"</span><span class="p">])</span>
        <span class="kt">XCTAssertEqual</span><span class="p">(</span><span class="n">controller</span><span class="o">.</span><span class="n">store</span><span class="o">.</span><span class="n">state</span><span class="o">.</span><span class="n">dataSource</span><span class="o">.</span><span class="n">todos</span><span class="p">,</span> <span class="p">[</span><span class="s">"2"</span><span class="p">,</span> <span class="s">"3"</span><span class="p">])</span>
    <span class="c1">// 现在 Command 只有 .loadToDos 一个命令。如果存在多个 Command，可以去下面的注释，</span>
    <span class="c1">// 这样在命令不符时可以让测试失败</span>
    <span class="c1">// default:</span>
    <span class="c1">//     XCTFail("The command should be .loadToDos")</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

<p>我们检查了返回的命令是否是 <code class="highlighter-rouge">.loadToDos</code>，而且 <code class="highlighter-rouge">.loadToDos</code> 的 <code class="highlighter-rouge">handler</code> 充当了天然的 stub。通过用一组 dummy 数据 (<code class="highlighter-rouge">["2", "3"]</code>) 进行调用，我们可以检查 <code class="highlighter-rouge">store</code> 中的状态是否如我们预期，这样我们就用同步的方式测试了异步加载的过程！</p>

<blockquote>
  <p>可能有同学会有疑问，认为这里没有测试 <code class="highlighter-rouge">ToDoStore.shared.getToDoItems</code>。但是记住，我们这里要测试的是 View Controller，而不是网络层。对于 <code class="highlighter-rouge">ToDoStore</code> 的测试应该放在单独的地方进行。</p>

  <p>你可以在 GitHub repo 的 <a href="https://github.com/onevcat/ToDoDemo/tree/reducer">reducer 分支</a>中找到对应这部分的代码。</p>
</blockquote>

<h2 id="总结">总结</h2>

<p>可能你已经见过类似的单向数据流的方式了，比如 <a href="https://github.com/reactjs/redux">Redux</a>，或者更古老一些的 <a href="http://facebook.github.io/flux/">Flux</a>。甚至在 Swift 中，也有 <a href="https://github.com/ReSwift/ReSwift">ReSwift</a> 实现了类似的想法。在这篇文章中，我们保持了基本的 MVC 架构，而使用了这种方法改进了 View Controller 的设计。</p>

<p>在例子中，我们的 <code class="highlighter-rouge">Store</code> 位于 View Controller 中。其实只要存在状态变化，这套方式可以在任何地方适用。你完全可以在其他的层级中引入 <code class="highlighter-rouge">Store</code>。只要能保证数据的单向流动，以及完整的状态变更覆盖测试，这套方式就具有良好的扩展性。</p>

<p>相对于大刀阔斧地改造，或者使用全新的设计模式，这种稍微小一些改进更容易在日常中进行探索和实践，它不存在什么外部依赖，可以被直接用在新建的 View Controller 中，你也可以逐步将已有类进行改造。毕竟绝大多数 iOS 开发者可能都会把大量时间花在 View Controller 上，所以能否写出易于测试，易于维护的 View Controller，多少将决定一个 iOS 开发者的幸福程度。所以花一些时间琢磨如何写好 View Controller，应该是每个 iOSer 的必修课。</p>

<h3 id="一些推荐的参考资料">一些推荐的参考资料</h3>

<p>如果你对函数式编程的一些概念感兴趣，不妨看看我和一些同仁翻译的<a href="https://objccn.io/products/functional-swift/">《函数式 Swift》</a>一书，里面对像是值类型、纯函数、引用透明等特性进行了详细的阐述。如果你想更多接触一些类似的架构方法，我个人推荐研读一下 <a href="https://facebook.github.io/react/">React</a> 的资料，特别是如何<a href="https://facebook.github.io/react/docs/thinking-in-react.html">以 React 的思想思考</a>的相关内容。如果你还有余力，即使你日常每天还是做 CocoaTouch 的 native 开发，也不妨尝试用 React Native 来构建一些项目。相信你会在这个过程中开阔眼界，得到新的领悟。</p>


  </section>



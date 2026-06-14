---
title: Home
layout: default
---

# <span data-en="Hi, I'm Brokensnow2" data-zh="你好，我是 Brokensnow2">Hi, I'm Brokensnow2</span>

<p data-en-html="I work on <strong>Vision-and-Language Navigation</strong>, <strong>Embodied AI</strong>, <strong>Robotics</strong>, and <strong>3D Perception</strong>. Outside research, I also like tinkering with embedded systems, music, and small game-dev ideas." data-zh-html="我现在主要做 <strong>视觉语言导航</strong>、<strong>具身智能</strong>、<strong>机器人</strong> 和 <strong>三维感知</strong> 相关的研究，也会折腾一些嵌入式、音乐和游戏开发的小东西。">I work on <strong>Vision-and-Language Navigation</strong>, <strong>Embodied AI</strong>, <strong>Robotics</strong>, and <strong>3D Perception</strong>. Outside research, I also like tinkering with embedded systems, music, and small game-dev ideas.</p>

<div class="bio-links">
  <a href="/about/" data-en="About" data-zh="关于">About</a>
  <a href="/projects/" data-en="Projects & Publications" data-zh="项目与论文">Projects & Publications</a>
  <a href="/posts/" data-en="Recent Posts" data-zh="近期文章">Recent Posts</a>
  <a href="/moments/" data-en="Moments" data-zh="动态">Moments</a>
  <a href="mailto:brokensnow2@gmail.com" data-en="Email" data-zh="邮件">Email</a>
  <a href="https://github.com/brokensnow2">GitHub</a>
  <a href="https://blog.csdn.net/qq_51503116?spm=1000.2115.3001.5343" target="_blank" rel="noopener">CSDN</a>
  <a href="https://soundcloud.com/violetdove" target="_blank" rel="noopener">SoundCloud</a>
</div>

---

## <span data-en="Research Interests" data-zh="研究兴趣">Research Interests</span>

<div class="interests-grid">
  <div class="interest-card">
    <span class="interest-icon">🧭</span>
    <strong>VLN</strong>
    <p data-en="Teaching agents to follow natural-language directions and move through 3D spaces." data-zh="研究怎么让机器人或智能体听懂自然语言，并在三维环境里走到目标位置。">Teaching agents to follow natural-language directions and move through 3D spaces.</p>
  </div>
  <div class="interest-card">
    <span class="interest-icon">🤖</span>
    <strong data-en="Embodied AI" data-zh="具身智能">Embodied AI</strong>
    <p data-en="Models that see, reason, and act in simulated or physical environments." data-zh="关心模型如何在真实或模拟环境里看、想、动，而不只是停留在屏幕上。">Models that see, reason, and act in simulated or physical environments.</p>
  </div>
  <div class="interest-card">
    <span class="interest-icon">🧊</span>
    <strong data-en="3D Perception" data-zh="三维感知">3D Perception</strong>
    <p data-en="Recovering structure and meaning from point clouds, depth, and multi-view images." data-zh="从点云、深度图和多视角图像里恢复空间结构，也理解场景里有什么。">Recovering structure and meaning from point clouds, depth, and multi-view images.</p>
  </div>
  <div class="interest-card">
    <span class="interest-icon">🔧</span>
    <strong data-en="Embedded Development" data-zh="嵌入式开发">Embedded Development</strong>
    <p data-en="Microcontrollers, sensors, low-level systems, and small hardware-software builds." data-zh="会玩一些单片机、传感器和底层系统，把代码接到真实硬件上。">Microcontrollers, sensors, low-level systems, and small hardware-software builds.</p>
  </div>
  <div class="interest-card">
    <span class="interest-icon">🎧</span>
    <strong data-en="Music" data-zh="音乐">Music</strong>
    <p data-en="Songs, sound design, and the things that do not quite belong in a paper." data-zh="写歌、做一点声音设计，也记录那些不太适合写成论文的东西。">Songs, sound design, and the things that do not quite belong in a paper.</p>
  </div>
  <div class="interest-card">
    <span class="interest-icon">🎮</span>
    <strong data-en="Game Dev" data-zh="游戏开发">Game Dev</strong>
    <p data-en="Procedural generation, real-time rendering, and playful systems that are fun to run." data-zh="喜欢程序生成、实时渲染，还有那种能马上跑起来的小玩法。">Procedural generation, real-time rendering, and playful systems that are fun to run.</p>
  </div>
</div>

---

## <span data-en="Recent Posts" data-zh="近期文章">Recent Posts</span>

{% if site.posts.size > 0 %}
<ul class="post-list">
  {% for post in site.posts limit:3 %}
    <li class="post-item">
      <a href="{{ post.url }}">{{ post.title }}</a>
      <span class="post-date">{{ post.date | date: "%Y-%m-%d" }}</span>
      {% if post.description %}
        <p class="post-excerpt">{{ post.description }}</p>
      {% elsif post.excerpt %}
        <p class="post-excerpt">{{ post.excerpt | strip_html | truncatewords: 30 }}</p>
      {% endif %}
    </li>
  {% endfor %}
</ul>
<p><a class="more-link" href="/posts/" data-en="View all posts" data-zh="看全部文章">View all posts</a></p>
{% else %}
<p data-en="No posts yet. I will add some here later." data-zh="还没开始写，之后慢慢补。">No posts yet. I will add some here later.</p>
{% endif %}

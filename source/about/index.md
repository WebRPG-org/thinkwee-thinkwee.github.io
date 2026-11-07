title: About Me
comments: false
html: true
---

<!-- Firebase SDK -->
<script src="https://www.gstatic.com/firebasejs/10.8.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.8.0/firebase-database-compat.js"></script>

<!-- Initialize Firebase and PV counter -->
<script>
  var prefersReducedMotion = window.matchMedia && window.matchMedia('(prefers-reduced-motion: reduce)').matches;

  // Count up from 0 to target with an easeOut curve; respects reduced-motion.
  function animateCount(el, target, duration, formatter) {
    formatter = formatter || function (v) { return String(Math.round(v)); };
    target = Number(target) || 0;
    if (prefersReducedMotion || target <= 0) {
      el.textContent = formatter(target);
      return;
    }
    var startTime = null;
    function step(now) {
      if (startTime === null) startTime = now;
      var p = Math.min((now - startTime) / duration, 1);
      var eased = 1 - Math.pow(1 - p, 3);
      el.textContent = formatter(eased * target);
      if (p < 1) {
        requestAnimationFrame(step);
      } else {
        el.textContent = formatter(target);
      }
    }
    requestAnimationFrame(step);
  }

  // Run cb once, the first time el scrolls into the viewport (immediately if unsupported).
  function whenVisible(el, cb) {
    if (!el || !('IntersectionObserver' in window)) { cb(); return; }
    var io = new IntersectionObserver(function (entries) {
      for (var i = 0; i < entries.length; i++) {
        if (entries[i].isIntersecting) {
          io.disconnect();
          cb();
          return;
        }
      }
    }, { threshold: 0.25 });
    io.observe(el);
  }

  // Wait for DOM to be fully loaded
  document.addEventListener('DOMContentLoaded', function() {
    // Your web app's Firebase configuration
    const firebaseConfig = {
      databaseURL: "https://blog-a38f5-default-rtdb.asia-southeast1.firebasedatabase.app",
      projectId: "blog-a38f5",
      apiKey: "AIzaSyDgs_qagScUwYJr0O1MUyc3EcWbhyWzoCw",
      authDomain: "blog-a38f5.firebaseapp.com",
      storageBucket: "blog-a38f5.appspot.com",
      messagingSenderId: "1091568780806",
      appId: "1:1091568780806:web:774c32a22ef91802d92587"
    };

    // Initialize Firebase
    firebase.initializeApp(firebaseConfig);

    // Get a reference to the page views - update to match your database structure
    const pvRef = firebase.database().ref('pageViews');

    // Display page views: count up from 0 once the counter is visible AND the real
    // committed value has arrived, then reflect later live updates directly.
    const pageViewsElement = document.getElementById('page-views');
    let pvValue = 0;
    let pvVisible = false;
    let pvStarted = false;

    function tryStartPv() {
      if (pvStarted || !pvVisible || !pvValue || !pageViewsElement) return;
      pvStarted = true;
      animateCount(pageViewsElement, pvValue, 6000);
    }

    whenVisible(pageViewsElement, function () { pvVisible = true; tryStartPv(); });

    // Increment, then animate to the COMMITTED server value. We must use the
    // transaction's completion snapshot, not `.on('value')`: the SDK fires an
    // optimistic local event (10000 default + 1 = 10001) before fetching the real
    // value, and animating to that stale number is what showed "10001".
    pvRef.transaction(
      (currentViews) => (currentViews || 10000) + 1,
      (error, committed, snapshot) => {
        if (snapshot) {
          pvValue = snapshot.val() || 0;
          tryStartPv();
        }
      }
    );

    // Reflect live updates from other visitors, but only after the count-up began.
    pvRef.on('value', (snapshot) => {
      if (!pvStarted) return;
      if (pageViewsElement) pageViewsElement.textContent = snapshot.val() || 0;
    });
  });

  document.addEventListener('DOMContentLoaded', function() {
    const newsList = document.querySelector('.news-list');
    if (newsList) {
        newsList.scrollTop = newsList.scrollHeight;
    }
  });

  document.addEventListener('DOMContentLoaded', function() {
    var elements = document.querySelectorAll('[data-repo]');
    if (!elements || !elements.length) return;

    function formatStars(count) {
      if (typeof count !== 'number' || isNaN(count)) return 'N/A';
      if (count >= 1000000) return (count / 1000000).toFixed(1).replace(/\.0$/, '') + 'M';
      if (count >= 1000) return (count / 1000).toFixed(1).replace(/\.0$/, '') + 'k';
      return String(Math.round(count));
    }

    // Render a star count into `el` as a link, counting up from 0 to the value.
    // `value` may be a number (animated) or a string like 'N/A' (shown as-is).
    function showStars(el, repo, value, statusClass, title) {
      var url = 'https://www.star-history.com/#' + repo + '&Date';
      var a = document.createElement('a');
      a.href = url;
      a.target = '_blank';
      a.className = 'dynamic-value';
      a.title = title;
      el.innerHTML = '';
      el.appendChild(a);
      el.classList.add(statusClass);
      if (typeof value === 'number') {
        a.textContent = formatStars(0);
        whenVisible(el, function () { animateCount(a, value, 4800, formatStars); });
      } else {
        a.textContent = value;
      }
    }

    // Use localStorage for persistent caching across page loads.
    // Stores raw star counts (v2); v1 stored pre-formatted strings, so the key bump drops it.
    var CACHE_KEY = 'github_stars_cache_v2';
    var CACHE_DURATION = 24 * 60 * 60 * 1000; // 24 hours in milliseconds
    
    function getCache() {
      try {
        var cached = localStorage.getItem(CACHE_KEY);
        if (cached) {
          var data = JSON.parse(cached);
          // Check if cache is still valid
          if (Date.now() - data.timestamp < CACHE_DURATION) {
            return data.repos;
          }
        }
      } catch (e) {
        console.warn('Failed to read cache:', e);
      }
      return {};
    }

    function setCache(cache) {
      try {
        localStorage.setItem(CACHE_KEY, JSON.stringify({
          timestamp: Date.now(),
          repos: cache
        }));
      } catch (e) {
        console.warn('Failed to save cache:', e);
      }
    }

    var cache = getCache();
    var pendingRequests = 0;
    var maxConcurrentRequests = 2; // Limit concurrent requests
    var requestQueue = [];

    function processQueue() {
      if (pendingRequests >= maxConcurrentRequests || requestQueue.length === 0) {
        return;
      }

      var task = requestQueue.shift();
      pendingRequests++;
      
      var repo = task.repo;
      var el = task.el;

      fetch('https://api.github.com/repos/' + repo)
        .then(function(res) { 
          if (res.status === 403) {
            throw new Error('Rate limit exceeded');
          }
          return res.json(); 
        })
        .then(function(data) {
          var count = (data && typeof data.stargazers_count === 'number') ? data.stargazers_count : null;
          if (count !== null) {
            cache[repo] = count;
            setCache(cache);
          }
          showStars(el, repo, count === null ? 'N/A' : count, 'github-stars-loaded', 'Live data from GitHub - Click to view star history');
        })
        .catch(function(err) {
          console.warn('Failed to fetch stars for ' + repo + ':', err.message);
          // Use fallback values if available
          var fallbackValue;
          if (repo === 'OpenBMB/ChatDev') {
            fallbackValue = 28000;
          } else if (repo === 'thinkwee/AgentsMeetRL') {
            fallbackValue = 490;
          } else {
            fallbackValue = 'N/A';
          }
          showStars(el, repo, fallbackValue, 'github-stars-fallback', 'Cached data (API rate limited) - Click to view star history');
        })
        .finally(function() {
          pendingRequests--;
          // Process next item in queue
          setTimeout(processQueue, 1000); // Add 1 second delay between requests
        });
    }

    elements.forEach(function(el) {
      var repo = el.getAttribute('data-repo');
      if (!repo) return;

      // Show loading state
      el.innerHTML = '<span class="dynamic-value loading">...</span>';

      // Check if we have cached value
      if (cache[repo] !== undefined) {
        showStars(el, repo, cache[repo], 'github-stars-cached', 'Cached data from GitHub - Click to view star history');
        return;
      }

      // Add to request queue
      requestQueue.push({ repo: repo, el: el });
    });

    // Start processing queue
    processQueue();
    if (requestQueue.length > 0) {
      setTimeout(processQueue, 1000);
    }
  });
</script>

<script>
  document.addEventListener('DOMContentLoaded', function () {
    // News: fade the top edge only while older items sit above.
    var list = document.querySelector('.news-list');
    if (list) {
      var sync = function () { list.classList.toggle('has-above', list.scrollTop > 4); };
      list.addEventListener('scroll', sync, { passive: true });
      setTimeout(sync, 50);
    }

    // Gentle fade-up for publication rows as they scroll into view.
    var reduce = window.matchMedia && window.matchMedia('(prefers-reduced-motion: reduce)').matches;
    if (reduce || !('IntersectionObserver' in window)) return;
    var rows = document.querySelectorAll('.pub, .feat-card, .about-social a');
    var io = new IntersectionObserver(function (entries) {
      entries.forEach(function (e) {
        if (!e.isIntersecting) return;
        var el = e.target;
        el.classList.add('is-in');
        io.unobserve(el);
        // once revealed, hand transitions back to the normal hover styles
        setTimeout(function () {
          el.classList.remove('will-reveal', 'is-in');
          el.style.transitionDelay = '';
        }, 1100);
      });
    }, { threshold: 0.1, rootMargin: '0px 0px -40px 0px' });
    rows.forEach(function (el, i) {
      var sib = el.parentElement ? Array.prototype.indexOf.call(el.parentElement.children, el) : 0;
      el.style.transitionDelay = Math.min(sib, 8) * 45 + 'ms';
      el.classList.add('will-reveal');
      io.observe(el);
    });
  });
</script>

<style>
    /* ---------- Page frame: comfortable measure on any screen ---------- */
    .post-body {
        container-type: inline-size;
        max-width: 960px;
        margin-left: auto;
        margin-right: auto;
    }
    .post-body ul,
    .post-body ol,
    .post-body li {
        text-align: left;
    }

    /* Scroll reveal (only applied when JS + IntersectionObserver are present) */
    .will-reveal {
        opacity: 0;
        transform: translateY(10px);
        transition: opacity .6s ease, transform .6s cubic-bezier(.2,.7,.2,1);
    }
    .will-reveal.is-in {
        opacity: 1;
        transform: none;
    }

    /* ---------- Hero ---------- */
    .about-hero { text-align: center; }
    .about-hero img {
        display: block;
        width: 100%;
        max-width: 600px;
        height: auto;
        margin: 0 auto 16px !important;
        border-radius: 12px;
        box-shadow: 0 10px 30px rgba(0,0,0,.08);
        transition: transform .6s cubic-bezier(.2,.7,.2,1), box-shadow .6s ease;
    }
    .about-hero img:hover {
        transform: translateY(-3px) scale(1.01);
        box-shadow: 0 16px 40px rgba(0,0,0,.12);
    }
    .about-hero p { text-align: center; margin: 0; line-height: 1.9; }
    .about-findme { margin-top: 22px !important; color: #888; }

    .about-social {
        display: flex;
        flex-wrap: wrap;
        justify-content: center;
        gap: 10px;
        margin: 12px 0 0;
    }
    .about-social > p { display: contents; }

    .libutton, .gsbutton, .ghbutton, .gmbutton, .aclbutton {
      display: inline-flex;
      align-items: center;
      justify-content: center;
      gap: 6px;
      padding: 0px 12px;
      text-align: center;
      outline: none;
      text-decoration: none !important;
      color: black !important;
      background-color: #FFFFFF;
      border: 1px solid #CCCCCC !important;
      border-radius: 8px;
      font-family: "SF Pro Text", -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      font-size: 14px;
      font-weight: 500;
      height: 32px;
      white-space: nowrap;
      transition: all 0.3s ease;
      cursor: pointer;
    }
    .libutton svg, .gsbutton svg, .ghbutton svg, .gmbutton svg, .aclbutton svg { flex-shrink: 0; }

    .libutton:hover, .gsbutton:hover, .ghbutton:hover, .gmbutton:hover, .aclbutton:hover {
      color: white !important;
      border: 1px solid transparent !important;
      transform: translateY(-2px);
      box-shadow: 0 8px 18px rgba(0,0,0,.15);
    }
    .libutton:hover  { background: linear-gradient(135deg, #8fa6c4 0%, #3d5470 100%); }
    .gsbutton:hover  { background: linear-gradient(135deg, #6090d4 0%, #2d4a8a 100%); }
    .ghbutton:hover  { background: linear-gradient(135deg, #706a90 0%, #2a2a2a 100%); }
    .gmbutton:hover  { background: linear-gradient(135deg, #d47570 0%, #a03838 100%); }
    .aclbutton:hover { background: linear-gradient(135deg, #f07a7f 0%, #c53a40 100%); }

    /* ---------- Logos ---------- */
    .about-logos {
        display: grid;
        grid-template-columns: repeat(3, minmax(0, 1fr));
        gap: 3%;
        max-width: 760px;
        margin: 0 auto 10px;
    }
    .about-logos > p { display: contents; }
    .about-logos img {
        display: block;
        width: 100%;
        height: auto;
        margin: 0 !important;
        border-radius: 6px;
        transition: transform .4s cubic-bezier(.2,.7,.2,1), box-shadow .4s ease;
    }
    .about-logos img:hover {
        transform: translateY(-4px);
        box-shadow: 0 10px 22px rgba(0,0,0,.12);
    }

    /* ---------- Journey: research interests as chapters ---------- */
    .journey { margin: 22px 0 6px; }
    .era {
        --c: #888;
        --g: linear-gradient(135deg, #bbb 0%, #666 100%);
        position: relative;
        padding: 0 0 22px 30px;
    }
    .era:last-child { padding-bottom: 0; }
    /* muted, matte tones: low saturation, gentle tonal gradients */
    .era-compress { --c: #8fa3b1; --g: linear-gradient(135deg, #adbdc8 0%, #8197a6 100%); }
    .era-agentic  { --c: #a597b3; --g: linear-gradient(135deg, #bfb3ca 0%, #9686a6 100%); }
    .era-wild     { --c: #98ad8c; --g: linear-gradient(135deg, #b5c6aa 0%, #8aa07d 100%); }

    /* the rail: a soft line linking the chapters */
    .era::before {
        content: "";
        position: absolute;
        left: 7px;
        top: 20px;
        bottom: -2px;
        width: 2px;
        border-radius: 1px;
        background: linear-gradient(to bottom, #e3e3e3, #eee);
    }
    .era:last-child::before { display: none; }

    .era-head {
        position: relative;
        display: flex;
        flex-wrap: wrap;
        align-items: center;
        gap: 6px 12px;
        margin-bottom: 6px;
    }
    /* chapter node */
    .era-head::before {
        content: "";
        position: absolute;
        left: -30px;
        top: 13px;
        width: 12px;
        height: 12px;
        margin-top: -8px;
        border-radius: 50%;
        background: #fff;
        border: 2px solid #ccc;
        transition: all .35s ease;
    }
    .era:hover .era-head::before { border-color: transparent; background: var(--g); transform: scale(1.15); }
    .era.is-now .era-head::before { border-color: transparent; background: var(--g); animation: era-pulse 2.4s ease-out infinite; }
    @keyframes era-pulse {
        0%   { box-shadow: 0 0 0 0 rgba(152,173,140,.5); }
        70%  { box-shadow: 0 0 0 9px rgba(152,173,140,0); }
        100% { box-shadow: 0 0 0 0 rgba(152,173,140,0); }
    }

    /* period badge — same vocabulary as the conference badges */
    .era-time {
        display: inline-block;
        padding: 0 8px;
        font-size: 14px;
        white-space: nowrap;
        background: #fff;
        color: black;
        border: 1px solid #ccc;
        border-radius: 8px;
        transition: all .3s ease;
        font-variant-numeric: tabular-nums;
    }
    .era:hover .era-time { background: var(--g); color: #fff; border-color: transparent; }
    .era-topic { font-weight: bold; color: #333; }
    .era-topic strong { font-weight: bold; }
    .era-note { font-weight: normal; color: #555; }

    .era-item {
        position: relative;
        display: grid;
        grid-template-columns: 6.6em minmax(0, 1fr);
        column-gap: 12px;
        align-items: baseline;
        padding: 4px 0;
        line-height: 1.8;
    }
    .era-item::before {
        content: "";
        position: absolute;
        left: -26px;
        top: 1.05em;
        width: 6px;
        height: 6px;
        border-radius: 50%;
        background: #d6d6d6;
        transition: background .3s ease, transform .3s ease;
    }
    .era-item:hover::before { background: var(--c); transform: scale(1.3); }
    .era-date {
        font-size: 13px;
        color: #888;
        font-variant-numeric: tabular-nums;
        line-height: 1.5;
    }
    .era-text { min-width: 0; text-align: left; }

    /* ---------- News ---------- */
    .news-list {
        max-height: 460px;
        overflow-y: auto;
        list-style: none;
        padding: 0 6px 0 0;
        margin: 0;
        overscroll-behavior: contain;
        scrollbar-width: thin;
        scrollbar-color: #ccc transparent;
    }
    .news-list.has-above {
        -webkit-mask-image: linear-gradient(to bottom, transparent 0, #000 48px);
                mask-image: linear-gradient(to bottom, transparent 0, #000 48px);
    }
    .news-list::-webkit-scrollbar { width: 4px; }
    .news-list::-webkit-scrollbar-track { background: transparent; }
    .news-list::-webkit-scrollbar-thumb { background: #ccc; }
    .news-list::-webkit-scrollbar-thumb:hover { background: #888; }

    .news-item {
        display: grid;
        grid-template-columns: auto 6.2em minmax(0, 1fr);
        column-gap: 12px;
        align-items: baseline;
        padding: 6px 0;
        margin: 0 !important;
        line-height: 1.8;
    }
    .news-item::before {
        content: "•";
        color: #888;
        transition: color .3s ease;
    }
    .news-item:hover::before { color: #6699FF; }
    .news-date {
        font-size: 13px;
        color: #888;
        font-family: inherit;
        font-variant-numeric: tabular-nums;
        white-space: nowrap;
    }
    .news-content {
        color: #555;
        font-size: 14px;
        text-align: left;
    }
    .news-content a {
        color: #555;
        text-decoration: none;
        border-bottom: 1px solid #ddd;
        transition: border-color .3s ease;
    }
    .news-content a:hover { border-bottom-color: #555; }
    .news-highlight {
        background: #f5f5f5;
        padding: 0 3px;
    }
    .news-content ul { margin: 4px 0 0 0; padding-left: 20px; }
    .news-content li { margin: 2px 0; }

    /* ---------- Selected papers: 3×3 grid ---------- */
    .feat-grid {
        display: grid;
        grid-template-columns: repeat(3, minmax(0, 1fr));
        gap: 28px 22px;
        margin: 18px 0 22px;
    }
    .post-body a.feat-card,
    .feat-card {
        display: flex;
        flex-direction: column;
        color: inherit;
        border-bottom: none !important;
        text-decoration: none !important;
    }
    .feat-cover {
        position: relative;
        aspect-ratio: 16 / 9;
        overflow: hidden;
        border-radius: 10px;
        background: #f3f0ea;
        box-shadow: 0 1px 2px rgba(0,0,0,.04);
        transition: box-shadow .5s ease, transform .5s cubic-bezier(.2,.7,.2,1);
    }
    .feat-cover img {
        display: block;
        width: 100%;
        height: 100%;
        object-fit: cover;
        margin: 0 !important;
        transition: transform .8s cubic-bezier(.2,.7,.2,1);
    }
    .feat-card:hover .feat-cover {
        transform: translateY(-3px);
        box-shadow: 0 14px 30px rgba(60,50,40,.13);
    }
    .feat-card:hover .feat-cover img { transform: scale(1.045); }
    .feat-meta {
        display: flex;
        align-items: center;
        gap: 6px;
        margin: 12px 0 2px;
        font-size: 12.5px;
        color: #999;
        letter-spacing: .02em;
    }
    .feat-meta::before {
        content: "";
        width: 6px;
        height: 6px;
        border-radius: 50%;
        background: var(--dot, #bbb);
    }
    .feat-title {
        text-align: left;
        font-weight: bold;
        color: #333;
        line-height: 1.5;
        transition: color .3s ease;
    }
    .feat-card:hover .feat-title { color: #6699FF; }
    .feat-desc {
        margin-top: 4px;
        font-size: 14px;
        line-height: 1.7;
        color: #777;
        text-align: left;
    }
    .dot-arxiv   { --dot: #c9a0aa; }
    .dot-emnlp   { --dot: #e3a3a5; }
    .dot-icml    { --dot: #9db7d6; }
    .dot-neurips { --dot: #b3a3c4; }
    .dot-acl     { --dot: #e3a3a5; }

    /* ---------- Full list, folded by default ---------- */
    .pub-fold { margin: 0; }
    .pub-fold > summary {
        list-style: none;
        display: flex;
        justify-content: center;
        margin: 4px 0 10px;
        cursor: pointer;
    }
    .pub-fold > summary::-webkit-details-marker { display: none; }
    .pub-fold-btn {
        display: inline-flex;
        align-items: center;
        gap: 8px;
        padding: 0 14px;
        height: 32px;
        font-size: 14px;
        color: black;
        background: #fff;
        border: 1px solid #ccc;
        border-radius: 8px;
        transition: all .3s ease;
        user-select: none;
    }
    .pub-fold-btn::after {
        content: "";
        width: 6px;
        height: 6px;
        border-right: 1.5px solid currentColor;
        border-bottom: 1.5px solid currentColor;
        transform: translateY(-2px) rotate(45deg);
        transition: transform .3s ease;
    }
    .pub-fold[open] .pub-fold-btn::after { transform: translateY(1px) rotate(-135deg); }
    .pub-fold > summary:hover .pub-fold-btn {
        background: #888;
        color: #fff;
        border-color: transparent;
        transform: translateY(-1px);
    }
    .pub-fold-open-text { display: none; }
    .pub-fold[open] .pub-fold-open-text { display: inline; }
    .pub-fold[open] .pub-fold-closed-text { display: none; }
    .pub-fold[open] > .pub-fold-body { animation: pub-unfold .45s ease both; }
    @keyframes pub-unfold {
        from { opacity: 0; transform: translateY(-6px); }
        to   { opacity: 1; transform: none; }
    }
    .pub-legend { color: #888; font-size: 14px; margin: 6px 0 4px; text-align: center; }
    .pub-legend span { white-space: nowrap; margin: 0 10px; }

    /* ---------- Publications ---------- */
    .pub-group-title {
        margin: 22px 0 8px;
        font-weight: bold;
        color: #333;
    }
    .pub-list { margin: 0 0 6px; }
    .pub {
        display: grid;
        grid-template-columns: 110px 52px 52px minmax(0, 1fr);
        grid-template-areas: "venue paper code title";
        column-gap: 6px;
        align-items: baseline;
        padding: 5px 8px;
        margin: 0 -8px;
        border-radius: 8px;
        transition: background-color .3s ease;
    }
    .pub:hover { background-color: #f7f8fa; }
    .pub > [class^="conf-"] { grid-area: venue; }
    .pub > .bp { grid-area: paper; }
    .pub > .bc { grid-area: code; }
    .pub > .pub-title { grid-area: title; line-height: 1.7; padding-left: 4px; text-align: left; }

    .bc, .bp {
        display: inline-block;
        padding: 0px 5px;
        font-size: 14px;
        text-align: center;
        text-decoration: none;
        background-color: #FFFFFF;
        color: black !important;
        border-radius: 8px;
        border: 1px solid #CCCCCC !important;
        transition: all 0.3s ease;
    }
    .bp:hover {
        background-color: #6699FF;
        color: white !important;
        border: 1px solid transparent !important;
        transform: translateY(-1px);
    }
    .bc:hover {
        background-color: #888888;
        color: white !important;
        border: 1px solid transparent !important;
        transform: translateY(-1px);
    }

    /* Conference badges — neutral; fill with the conference color on hover
       (of the badge itself, or of the whole publication row) */
    [class^="conf-"] {
        display: inline-block;
        padding: 0px 5px;
        font-size: 14px;
        text-align: center;
        text-decoration: none;
        white-space: nowrap;
        background-color: #FFFFFF !important;
        color: black !important;
        border-radius: 8px;
        border: 1px solid #CCCCCC;
        transition: all 0.3s ease;
        cursor: default;
    }
    [class^="conf-"]:hover,
    .pub:hover > [class^="conf-"] {
        color: white !important;
        border: 1px solid transparent;
    }
    .conf-neurips:hover, .pub:hover > .conf-neurips { background-color: #8668a3 !important; }
    .conf-iclr:hover,    .pub:hover > .conf-iclr    { background-color: #6fca62 !important; }
    .conf-icml:hover,    .pub:hover > .conf-icml    { background-color: #4a90e2 !important; }
    .conf-acl:hover,     .pub:hover > .conf-acl     { background-color: #f14950 !important; }
    .conf-aaai:hover,    .pub:hover > .conf-aaai    { background-color: #5b90a8 !important; }
    .conf-conll:hover,   .pub:hover > .conf-conll   { background-color: #f14950 !important; }
    .conf-emnlp:hover,   .pub:hover > .conf-emnlp   { background-color: #f14950 !important; }
    .conf-sigir:hover,   .pub:hover > .conf-sigir   { background-color: #e6a800 !important; }
    .conf-ranlp:hover,   .pub:hover > .conf-ranlp   { background-color: #f14950 !important; }
    .conf-ccis:hover,    .pub:hover > .conf-ccis    { background-color: #f14950 !important; }
    .conf-arxiv:hover,   .pub:hover > .conf-arxiv   { background-color: #c45569 !important; }

    /* ---------- Social media ---------- */
    .sm-grid {
        display: grid;
        grid-template-columns: repeat(2, minmax(0, 1fr));
        gap: 22px;
        margin: 18px 0 18px;
    }
    .sm-card {
        --dot: #bbb;
        display: flex;
        align-items: center;
        gap: 18px;
        padding: 16px 18px;
        background: #fff;
        border: 1px solid #ebe7e1;
        border-radius: 12px;
        transition: transform .5s cubic-bezier(.2,.7,.2,1), box-shadow .5s ease, border-color .5s ease;
    }
    .sm-card:hover {
        transform: translateY(-3px);
        border-color: transparent;
        box-shadow: 0 14px 30px rgba(60,50,40,.12);
    }
    .sm-wechat  { --dot: #9cb9a2; --hl: #07c160; }
    .sm-rednote { --dot: #d6a3a6; --hl: #ff2442; }
    .sm-qr {
        flex: 0 0 auto;
        width: 120px;
        height: 120px;
        margin: 0 !important;
        border-radius: 8px;
    }
    .sm-text { min-width: 0; text-align: left; }
    .sm-meta {
        display: flex;
        align-items: center;
        gap: 6px;
        font-size: 12.5px;
        color: #999;
        letter-spacing: .02em;
    }
    .sm-meta::before {
        content: "";
        width: 6px;
        height: 6px;
        border-radius: 50%;
        background: var(--dot);
        transition: background .4s ease, transform .4s ease;
    }
    .sm-card:hover .sm-meta::before { background: var(--hl); transform: scale(1.3); }
    .sm-name { margin-top: 2px; font-weight: bold; color: #333; line-height: 1.5; }
    .sm-desc { margin-top: 4px; font-size: 14px; line-height: 1.7; color: #777; }
    .sm-note { text-align: left; color: #555; }

    /* ---------- Live numbers ---------- */
    .pv-counter {
        display: inline;
        padding: 0;
        font-size: inherit;
        color: inherit;
        background: none;
        border: none;
    }
    .pv-counter #page-views { transition: color 0.3s ease; }
    .pv-counter:hover #page-views { color: #6699FF; }

    .dynamic-value {
        display: inline;
        font-family: 'SF Mono', 'Monaco', 'Consolas', monospace;
        border-bottom: 1px dotted #ccc;
        color: inherit;
        text-decoration: none;
    }
    a.dynamic-value { cursor: pointer; }
    span.dynamic-value.loading { cursor: default; }

    /* ---------- Narrow screens ---------- */
    @container (max-width: 600px) {
        .pub {
            grid-template-columns: auto auto auto 1fr;
            grid-template-areas:
                "venue paper code ."
                "title title title title";
            row-gap: 4px;
            padding: 8px 8px;
        }
        .pub > .pub-title { padding-left: 0; }
        .news-item {
            grid-template-columns: auto minmax(0, 1fr);
            grid-template-areas: "dot date" ". text";
        }
        .news-item::before { grid-area: dot; }
        .news-date { grid-area: date; }
        .news-content { grid-area: text; }
        .libutton, .gsbutton, .ghbutton, .gmbutton, .aclbutton {
            padding: 0px 10px;
            font-size: 13px;
            height: 30px;
        }
        .about-social { gap: 8px; }
        .era-item { grid-template-columns: minmax(0, 1fr); row-gap: 0; }
        .era-date br { display: none; }
    }
    @container (max-width: 760px) {
        .feat-grid { gap: 22px 14px; }
        .feat-desc { font-size: 13px; }
    }
    /* phones: compact rows, thumbnail left */
    @container (max-width: 540px) {
        .sm-grid { grid-template-columns: minmax(0, 1fr); gap: 14px; }
        .sm-qr { width: 104px; height: 104px; }
        .feat-grid { grid-template-columns: minmax(0, 1fr); gap: 16px; }
        .post-body a.feat-card {
            display: grid;
            grid-template-columns: 38% minmax(0, 1fr);
            grid-template-areas: "cover meta" "cover title" "cover desc";
            grid-template-rows: auto auto 1fr;
            column-gap: 12px;
            align-items: start;
        }
        .feat-cover { grid-area: cover; border-radius: 8px; }
        .feat-meta  { grid-area: meta; margin: 0; font-size: 12px; }
        .feat-title { grid-area: title; font-size: 15px; }
        .feat-desc  { grid-area: desc; margin-top: 2px; font-size: 13px; line-height: 1.6; }
    }
    @media (prefers-reduced-motion: reduce) {
        .post-body *, .post-body *::before { transition: none !important; animation: none !important; }
    }
</style>

<div class="about-hero">
<img src="/img/bg_blog.png" alt="I'm in London">
<p>Fun Fact: My giraffe icon isn't random - I actually have a long neck!</p>
<p>See for yourself and meet other talented researchers on our <a href="https://kclnlp.github.io/team.html" target="_blank" rel="noopener">KCLNLP team page</a>.</p>
</div>

---

<div class="about-hero">
<p>You can find this guy via</p>
<div class="about-social">
<a class="aclbutton" href="https://aclanthology.org/people/wei-liu-kcl/" target="_blank"><svg width="16" height="16" viewBox="0 0 24 24" fill="currentColor"><path d="M19 2H8a2 2 0 0 0-2 2v1H5a2 2 0 0 0-2 2v11a2 2 0 0 0 2 2h11a2 2 0 0 0 2-2v-1h1a2 2 0 0 0 2-2V4a2 2 0 0 0-2-2Zm-3 16a1 1 0 0 1-1 1H5V7h10a1 1 0 0 1 1 1v10Zm3-3h-1V8a3 3 0 0 0-3-3H8V4h11v11Z"/></svg>ACL Anthology</a> <a class="libutton" href="http://www.linkedin.com/comm/mynetwork/discovery-see-all?usecase=PEOPLE_FOLLOWS&followMember=thinkwee" target="_blank"><svg width="16" height="16" viewBox="0 0 24 24" fill="currentColor"><path d="M20.447 20.452h-3.554v-5.569c0-1.328-.027-3.037-1.852-3.037-1.853 0-2.136 1.445-2.136 2.939v5.667H9.351V9h3.414v1.561h.046c.477-.9 1.637-1.85 3.37-1.85 3.601 0 4.267 2.37 4.267 5.455v6.286zM5.337 7.433c-1.144 0-2.063-.926-2.063-2.065 0-1.138.92-2.063 2.063-2.063 1.14 0 2.064.925 2.064 2.063 0 1.139-.925 2.065-2.064 2.065zm1.782 13.019H3.555V9h3.564v11.452zM22.225 0H1.771C.792 0 0 .774 0 1.729v20.542C0 23.227.792 24 1.771 24h20.451C23.2 24 24 23.227 24 22.271V1.729C24 .774 23.2 0 22.222 0h.003z"/></svg>Linkedin</a> <a class="gsbutton" href="https://scholar.google.com/citations?view_op=list_works&hl=en&user=QvW2leIAAAAJ" target="_blank"><svg width="16" height="16" viewBox="0 0 24 24" fill="currentColor"><path d="M5.242 13.769L0 9.5 12 0l12 9.5-5.242 4.269C17.548 11.249 14.978 9.5 12 9.5c-2.977 0-5.548 1.748-6.758 4.269zM12 10a7 7 0 1 0 0 14 7 7 0 0 0 0-14z"/></svg>Google Scholar</a> <a class="ghbutton" href="https://github.com/thinkwee" target="_blank"><svg width="16" height="16" viewBox="0 0 24 24" fill="currentColor"><path d="M12 0C5.374 0 0 5.373 0 12 0 17.302 3.438 21.8 8.207 23.387c.599.111.793-.261.793-.577v-2.234c-3.338.726-4.033-1.416-4.033-1.416-.546-1.387-1.333-1.756-1.333-1.756-1.089-.745.083-.729.083-.729 1.205.084 1.839 1.237 1.839 1.237 1.07 1.834 2.807 1.304 3.492.997.107-.775.418-1.305.762-1.604-2.665-.305-5.467-1.334-5.467-5.931 0-1.311.469-2.381 1.236-3.221-.124-.303-.535-1.524.117-3.176 0 0 1.008-.322 3.301 1.23A11.509 11.509 0 0 1 12 5.803c1.02.005 2.047.138 3.006.404 2.291-1.552 3.297-1.23 3.297-1.23.653 1.653.242 2.874.118 3.176.77.84 1.235 1.911 1.235 3.221 0 4.609-2.807 5.624-5.479 5.921.43.372.823 1.102.823 2.222v3.293c0 .319.192.694.801.576C20.566 21.797 24 17.3 24 12c0-6.627-5.373-12-12-12z"/></svg>GitHub</a> <a class="gmbutton" href="mailto:thinkwee2767@gmail.com"><svg width="16" height="16" viewBox="0 0 24 24" fill="currentColor"><path d="M24 5.457v13.909c0 .904-.732 1.636-1.636 1.636h-3.819V11.73L12 16.64l-6.545-4.91v9.273H1.636A1.636 1.636 0 0 1 0 19.366V5.457c0-.904.732-1.636 1.636-1.636h3.819v.273L12 10.186l6.545-6.092v-.273h3.819c.904 0 1.636.732 1.636 1.636Z"/></svg>Gmail</a>
</div>
</div>


# About Me

<div class="about-logos">
<img src="/img/bupt.png" alt="I'm in BUPT">
<img src="/img/tencent.png" alt="I'm in Tecent">
<img src="/img/tsinghua.png" alt="I'm in Tsinghua">
</div>

<div>
<ul>
<li>Hello, I'm Wei Liu (刘维). Welcome to my blog <span class="pv-counter">(<span id="page-views">0</span> views)</span>. You can search me on google with keyword "thinkwee", which means "The Thinking Wei".</li>
</ul>
</div>

<div class="journey">
<div class="era era-compress">
<div class="era-head"><span class="era-time">before 2023</span><span class="era-topic">Natural Language Processing</span></div>
<div class="era-item"><span class="era-date">2014 – 2021</span><div class="era-text">Bachelor and Master in BUPT, where I worked on abstractive summarization: improving both the diversity and the quality of generated summaries.</div></div>
<div class="era-item"><span class="era-date">2021 – 2023</span><div class="era-text">Application Research in <a href="https://www.tencent.com/en-us/about.html">Tencent</a>: 3 happy years bringing NLP into the wild of Recommendation &amp; Advertisement, from news-feed NLU and WeChat Ads to large-scale recommender systems.</div></div>
</div>
<div class="era era-agentic">
<div class="era-head"><span class="era-time">2023 – 2025</span><span class="era-topic">LLM Multi-Agent System</span></div>
<div class="era-item"><span class="era-date">2023.10</span><div class="era-text">I had heart surgery and replaced parts of my heart. Now I am the Iron Man!</div></div>
<div class="era-item"><span class="era-date">2023 – 2025</span><div class="era-text">Working at <a href="https://nlp.csai.tsinghua.edu.cn/">THUNLP</a>@Tsinghua University with Prof. <a href="http://nlp.csai.tsinghua.edu.cn/~lzy/">Zhiyuan Liu</a> and Prof. <a href="http://qianc62.github.io">Chen Qian</a> on LLM Multi-Agent System, including the popular multi-agent framework for software development, <a href="https://github.com/OpenBMB/ChatDev" target="_blank" rel="noopener"><strong>ChatDev</strong></a>.</div></div>
</div>
<div class="era era-wild is-now">
<div class="era-head"><span class="era-time">2025 – now</span><span class="era-topic"><strong>Agentic AI in the Wild</strong> <span class="era-note">(check out this <a href="https://thinkwee.top/2025/10/05/wild-era/#more">blog</a>)</span></span></div>
<div class="era-item"><span class="era-date">2025.2 - now</span><div class="era-text">Happy to join <a href="https://kclnlp.github.io/">KCLNLP</a> as a PhD candidate advised by <a href="https://sites.google.com/view/yulanhe">Prof. Yulan He</a> and <a href="https://yalidu.github.io/">Prof. Yali Du</a>! I am interested in developing Agentic AI in the real world and beyond the easy to verify domains.</div></div>
</div>
</div>

# Recent News

<ul class="news-list">
<li class="news-item"><span class="news-date">2023.09.07</span><span class="news-content">🚀 <a href="https://github.com/OpenBMB/ChatDev" target="_blank">ChatDev</a> made #1 on Github Trending! It now has earn over <span class="news-highlight"><span id="chatdev-stars" data-repo="OpenBMB/ChatDev">...</span> stars!</span></span></li>
<li class="news-item"><span class="news-date">2024.05.15</span><span class="news-content">🇹🇭 2 papers about LLM Multi-Agent System (<a href="https://arxiv.org/abs/2307.07924" target="_blank">ChatDev</a> and <a href="https://arxiv.org/abs/2312.17025" target="_blank">Experiential Co-Learning</a>) were accepted by <span class="news-highlight">ACL 2024</span>!</span></li>
<li class="news-item"><span class="news-date">2024.08.15</span><span class="news-content">🚀 I built an interactive site on popular MultiAgent frameworks and THUNLP MultiAgent Team works: <a href="https://thinkwee.top/multiagent_ebook" target="_blank">LLM MultiAgent EBook</a></span></li>
<li class="news-item"><span class="news-date">2024.09.26</span><span class="news-content">🇨🇦 1 paper about <a href="https://arxiv.org/abs/2406.14928" target="_blank">Personal Agentic AI</a> was accepted by <span class="news-highlight">NeurIPS 2024</span>!</span></li>
<li class="news-item"><span class="news-date">2025.01.22</span><span class="news-content">🇸🇬 1 paper about <a href="https://arxiv.org/abs/2406.07155" target="_blank">Scaling LLM Agents</a> was accepted by <span class="news-highlight">ICLR 2025</span>!</span></li>
<li class="news-item"><span class="news-date">2025.05.15</span><span class="news-content">🇦🇹 1 paper about <a href="https://arxiv.org/abs/2406.08979" target="_blank">LLM Agent for Software Development</a> was accepted by <span class="news-highlight">ACL 2025</span>!</span></li>
<li class="news-item"><span class="news-date">2025.02.21</span><span class="news-content">🇬🇧 I started my journey as a PhD student in the UK!</span></li>
<li class="news-item"><span class="news-date">2025.06.09</span><span class="news-content">🚀 I made <a href="https://thinkwee.top/amr/" target="_blank">AgentsMeetRL</a>, an awesome list of RL for training LLM Agents (<span class="news-highlight"><span id="agentsmeetrl-stars" data-repo="thinkwee/AgentsMeetRL">...</span> stars!</span>) welcome to the era of experience!</span></li>
<li class="news-item"><span class="news-date">2025.08.20</span><span class="news-content">🇨🇳 2 papers accepted by <span class="news-highlight">EMNLP 2025</span>, Check out <a href="https://arxiv.org/abs/2505.06904" target="_blank">EcoLANG</a> and <a href="https://arxiv.org/abs/2505.16022" target="_blank">NOVER</a>!</span></li>
<li class="news-item"><span class="news-date">2026.01.25</span><span class="news-content">🇧🇷 1 paper on <a href="https://openreview.net/forum?id=yz7fL5vfpn" target="_blank">Benchmarking LLM Tool-Use in the Wild</a> accepted by <span class="news-highlight">ICLR 2026</span>!</span></li>
<li class="news-item"><span class="news-date">2026.04.22</span><span class="news-content">🚀 We launched <a href="https://github.com/thinkwee/HiMe" target="_blank">HiMe</a>, a self-hosted, fully local, open-source one-stop personal health AI agent platform (<span class="news-highlight"><span id="hime-stars" data-repo="thinkwee/HiMe">...</span> stars!</span>), now accepted by <span class="news-highlight">EMNLP 2026 Demo</span> (<a href="https://arxiv.org/abs/2607.21019" target="_blank">paper</a>)!</span></li>
<li class="news-item"><span class="news-date">2026.04.28</span><span class="news-content">🚀 I made <a href="https://github.com/thinkwee/AwesomeOPD" target="_blank">AwesomeOPD</a>, an awesome list of on-policy distillation (<span class="news-highlight"><span id="awesomeopd-stars" data-repo="thinkwee/AwesomeOPD">...</span> stars!</span>)</span></li>
<li class="news-item"><span class="news-date">2026.04.30</span><span class="news-content">🇰🇷 3 papers accepted by <span class="news-highlight">ICML 2026</span>, on <a href="https://arxiv.org/abs/2602.02039" target="_blank">Deep Data Research</a>, <a href="https://arxiv.org/abs/2603.02218" target="_blank">self-evolving agents</a>, and <a href="https://arxiv.org/abs/2602.18145" target="_blank">hallucination detection</a>!</span></li>
<li class="news-item"><span class="news-date">2026.06.22</span><span class="news-content">🛡️ 2 new papers on <span class="news-highlight">AI Safety in the Wild</span>!<ul><li><a href="https://arxiv.org/abs/2606.04075" target="_blank">LLMs Hack Rewards, and Society</a> (featured in the news of <a href="https://www.science.org/content/article/ai-models-have-troubling-knack-discovering-legal-loopholes" target="_blank">Science</a>): RL-trained LLMs exploit gaps in societal rules.</li><li><a href="https://arxiv.org/abs/2606.09890" target="_blank">PreAct-Bench</a>: a benchmark of 1,000 paired trajectories testing whether agents can foresee unethical outcomes before they happen.</li></ul></span></li>
<li class="news-item"><span class="news-date">2026.11.07</span><span class="news-content">🎆 2 new papers on <span class="news-highlight">Agent Environment and Efficiency!</span>!<ul><li><a href="https://arxiv.org/abs/2610.33817" target="_blank">EnvLunch</a>: Agentic LLM can be learned from environment itself, you don't need to imitate trajectory or write tasks.</li><li><a href="https://arxiv.org/abs/2611.06976" target="_blank">PLatour</a>: Large Model prefill, small model decode, better performance-efficiency pareto.</li></ul></span></li>
</ul>

# Publications

<div class="feat-grid">
<a class="feat-card" href="https://arxiv.org/abs/2611.06976" target="_blank" rel="noopener"><div class="feat-cover"><img src="/img/cards/pact.webp" alt="PLatour" loading="lazy"></div><div class="feat-meta dot-arxiv">arXiv 2026</div><div class="feat-title">PLatour</div><div class="feat-desc">Large model prefills, small model decodes, for a better performance–efficiency Pareto.</div></a>
<a class="feat-card" href="https://arxiv.org/abs/2610.33817" target="_blank" rel="noopener"><div class="feat-cover"><img src="/img/cards/envlunch.webp" alt="EnvLunch" loading="lazy"></div><div class="feat-meta dot-arxiv">arXiv 2026</div><div class="feat-title">EnvLunch</div><div class="feat-desc">Agentic LLMs can learn from the environment itself, no imitated trajectories or hand-written tasks needed.</div></a>
<a class="feat-card" href="https://arxiv.org/abs/2607.21019" target="_blank" rel="noopener"><div class="feat-cover"><img src="/img/cards/hime.webp" alt="HiMe" loading="lazy"></div><div class="feat-meta dot-emnlp">EMNLP 2026 Demo</div><div class="feat-title">HiMe</div><div class="feat-desc">A real-time, self-hosted personal agent platform that turns wearable data into health insights.</div></a>
<a class="feat-card" href="https://arxiv.org/abs/2606.04075" target="_blank" rel="noopener"><div class="feat-cover"><img src="/img/cards/sociohack.webp" alt="SocioHack" loading="lazy"></div><div class="feat-meta dot-arxiv">arXiv 2026</div><div class="feat-title">SocioHack</div><div class="feat-desc">RL-trained LLMs exploit gaps in societal rules, just like they hack reward functions.</div></a>
<a class="feat-card" href="https://arxiv.org/abs/2603.02218" target="_blank" rel="noopener"><div class="feat-cover"><img src="/img/cards/self_play_position.webp" alt="Self-Play" loading="lazy"></div><div class="feat-meta dot-icml">ICML 2026</div><div class="feat-title">When Does Self-Play Evolve?</div><div class="feat-desc">Only when the self-synthetic pipeline ensures learnable information gain.</div></a>
<a class="feat-card" href="https://arxiv.org/abs/2602.02039" target="_blank" rel="noopener"><div class="feat-cover"><img src="/img/cards/ddrbench.webp" alt="DDR-Bench" loading="lazy"></div><div class="feat-meta dot-icml">ICML 2026</div><div class="feat-title">DDR-Bench</div><div class="feat-desc">Can LLMs hunt for insights in data on their own, instead of waiting for questions?</div></a>
<a class="feat-card" href="https://arxiv.org/abs/2505.16022" target="_blank" rel="noopener"><div class="feat-cover"><img src="/img/cards/nover.webp" alt="NOVER" loading="lazy"></div><div class="feat-meta dot-emnlp">EMNLP 2025</div><div class="feat-title">NOVER</div><div class="feat-desc">Verifier-free incentive training that brings RL reasoning to general text-to-text tasks.</div></a>
<a class="feat-card" href="https://arxiv.org/abs/2406.14928" target="_blank" rel="noopener"><div class="feat-cover"><img src="/img/cards/iagents.webp" alt="iAgents" loading="lazy"></div><div class="feat-meta dot-neurips">NeurIPS 2024</div><div class="feat-title">iAgents</div><div class="feat-desc">Agents collaborate on behalf of humans, while each holds only part of the information.</div></a>
<a class="feat-card" href="https://arxiv.org/abs/2307.07924" target="_blank" rel="noopener"><div class="feat-cover"><img src="/img/cards/chatdev_new.webp" alt="ChatDev" loading="lazy"></div><div class="feat-meta dot-acl">ACL 2024</div><div class="feat-title">ChatDev</div><div class="feat-desc">A virtual software company where communicative LLM agents design, code and test together.</div></a>
</div>

<details class="pub-fold">
<summary><span class="pub-fold-btn"><span class="pub-fold-closed-text">Show all publications</span><span class="pub-fold-open-text">Hide full list</span></span></summary>
<div class="pub-fold-body">
<div class="pub-legend"><span>♦️ denotes first/co-first author.</span> <span>♣️ denotes workshop/findings papers.</span></div>

<div class="pub-group-title">Agentic AI in the Wild</div>
<div class="pub-list">
<div class="pub"><span class="conf-emnlp">EMNLP 2026</span><a href="https://arxiv.org/abs/2607.21019" class="bp">paper</a><a href="https://github.com/thinkwee/HiMe" class="bc">code</a><span class="pub-title">HiMe: Real-Time Self-Hosted Personal Agent Platform for Health Insights with Wearable Devices♦️</span></div>
<div class="pub"><span class="conf-arxiv">arXiv 2026</span><a href="https://arxiv.org/abs/2610.33817" class="bp">paper</a><a href="https://github.com/thinkwee/EnvLunch" class="bc">code</a><span class="pub-title">EnvLunch: The Environment as a Free Lunch for Learning to Interact♦️</span></div>
<div class="pub"><span class="conf-arxiv">arXiv 2026</span><a href="https://arxiv.org/abs/2606.04075" class="bp">paper</a><a href="https://github.com/thinkwee/SocioHack" class="bc">code</a><span class="pub-title">Large Language Models Hack Rewards, and Society♦️</span></div>
<div class="pub"><span class="conf-iclr">ICLR 2026</span><a href="https://openreview.net/forum?id=yz7fL5vfpn" class="bp">paper</a><a href="https://github.com/yupeijei1997/WildToolBench" class="bc">code</a><span class="pub-title">Benchmarking LLM Tool-Use in the Wild</span></div>
<div class="pub"><span class="conf-icml">ICML 2026</span><a href="https://arxiv.org/abs/2602.02039" class="bp">paper</a><a href="https://github.com/thinkwee/DDR_Bench" class="bc">code</a><span class="pub-title">Hunt Instead of Wait: Evaluating Deep Data Research on Large Language Models♦️</span></div>
<div class="pub"><span class="conf-icml">ICML 2026</span><a href="https://arxiv.org/abs/2603.02218" class="bp">paper</a><a href="https://github.com/thinkwee/SelfPlay_SelfEvo_Gap" class="bc">code</a><span class="pub-title">Self-Play Only Evolves When Self-Synthetic Pipeline Ensures Learnable Information Gain♦️</span></div>
<div class="pub"><span class="conf-emnlp">EMNLP 2025</span><a href="https://arxiv.org/abs/2505.16022" class="bp">paper</a><a href="https://github.com/thinkwee/NOVER" class="bc">code</a><span class="pub-title">NOVER: Incentive Training for Language Models via Verifier-Free Reinforcement Learning♦️</span></div>
<div class="pub"><span class="conf-neurips">NeurIPS 2024</span><a href="https://arxiv.org/abs/2406.14928" class="bp">paper</a><a href="https://github.com/thinkwee/iAgents" class="bc">code</a><span class="pub-title">Autonomous Agents for Collaborative Task under Information Asymmetry♦️</span></div>
</div>

<div class="pub-group-title">LLM-based Multi-Agents System</div>
<div class="pub-list">
<div class="pub"><span class="conf-iclr">ICLR 2025</span><a href="https://arxiv.org/pdf/2406.07155" class="bp">paper</a><a href="https://github.com/OpenBMB/ChatDev" class="bc">code</a><span class="pub-title">Scaling Large-Language-Model-based Multi-Agent Collaboration</span></div>
<div class="pub"><span class="conf-emnlp">EMNLP 2025</span><a href="https://arxiv.org/pdf/2505.06904" class="bp">paper</a><a href="https://github.com/xymou/EcoLANG" class="bc">code</a><span class="pub-title">EcoLANG: Efficient and Effective Agent Communication Language Induction for Social Simulation♣️</span></div>
<div class="pub"><span class="conf-acl">ACL 2024</span><a href="https://arxiv.org/abs/2307.07924" class="bp">paper</a><a href="https://github.com/OpenBMB/ChatDev" class="bc">code</a><span class="pub-title">Communicative Agents for Software Development</span></div>
<div class="pub"><span class="conf-acl">ACL 2024</span><a href="https://arxiv.org/abs/2312.17025" class="bp">paper</a><a href="https://github.com/OpenBMB/ChatDev" class="bc">code</a><span class="pub-title">Experiential Co-Learning of Software-Developing Agents</span></div>
</div>

<div class="pub-group-title">Compression Intelligence in NLP</div>
<div class="pub-list">
<div class="pub"><span class="conf-acl">ACL 2021</span><a href="https://arxiv.org/pdf/2106.04847.pdf" class="bp">paper</a><a href="https://github.com/thinkwee/UniKeyphrase" class="bc">code</a><span class="pub-title">UniKeyphrase: A Unified Extraction and Generation Framework for Keyphrase Prediction♦️♣️</span></div>
<div class="pub"><span class="conf-aaai">AAAI 2022</span><a href="https://ojs.aaai.org/index.php/AAAI/article/download/21402/version/19689/21151" class="bp">paper</a><a href="https://github.com/m1594730237/FastAndConstrainedKeyphrase" class="bc">code</a><span class="pub-title">Fast and Constrained Absent Keyphrase Generation by Prompt-Based Learning</span></div>
<div class="pub"><span class="conf-conll">CoNLL 2021</span><a href="https://www.aclweb.org/anthology/K19-1077/" class="bp">paper</a><a href="https://github.com/thinkwee/DPP_CNN_Summarization" class="bc">code</a><span class="pub-title">Comprehensive Abstractive Summarization with Diversified Attention Based on Determinantal Point Processes♦️</span></div>
<div class="pub"><span class="conf-sigir">SIGIR 2019</span><a href="http://ceur-ws.org/Vol-2414/paper20.pdf" class="bp">paper</a><span class="pub-title">CIST@CLSciSumm-19: Automatic Scientific Paper Summarization with Citances and Facets♣️</span></div>
<div class="pub"><span class="conf-emnlp">EMNLP 2020</span><a href="https://www.aclweb.org/anthology/2020.sdp-1.25.pdf" class="bp">paper</a><span class="pub-title">CIST@CL-SciSumm 2020, LongSumm 2020: Automatic Scientific Document Summarization♦️♣️</span></div>
<div class="pub"><span class="conf-ranlp">RANLP 2019</span><a href="https://www.aclweb.org/anthology/W19-8904.pdf" class="bp">paper</a><a href="https://github.com/thinkwee/multiling2019_wiki" class="bc">code</a><span class="pub-title">Multi-lingual Wikipedia Summarization and Title Generation On Low Resource Corpus♦️♣️</span></div>
<div class="pub"><span class="conf-arxiv">arXiv 2021</span><a href="https://arxiv.org/pdf/2112.01147.pdf" class="bp">paper</a><a href="https://github.com/thinkwee/co2sum" class="bc">code</a><span class="pub-title">CO2Sum: Contrastive Learning for Factual-Consistent Abstractive Summarization♦️</span></div>
<div class="pub"><span class="conf-arxiv">arXiv 2021</span><a href="https://arxiv.org/pdf/2106.10084.pdf" class="bp">paper</a><a href="https://github.com/thinkwee/SubjectiveBiasABS" class="bc">code</a><span class="pub-title">Subjective Bias in Abstractive Summarization♦️</span></div>
</div>

<div class="pub-group-title">Collaborations (I am interested in many things!)</div>
<div class="pub-list">
<div class="pub"><span class="conf-arxiv">arXiv 2026</span><a href="https://arxiv.org/abs/2611.06976" class="bp">paper</a><a href="https://github.com/zhenyi4/platour" class="bc">code</a><span class="pub-title">On the Opportunities and Challenges of Prefill-to-Autoregressive Cross-Model Transfer</span></div>
<div class="pub"><span class="conf-arxiv">arXiv 2026</span><a href="https://arxiv.org/abs/2606.09890" class="bp">paper</a><a href="https://github.com/oyarsa/preact-bench" class="bc">code</a><span class="pub-title">PreAct-Bench: Benchmarking Predictive Monitoring in LLMs</span></div>
<div class="pub"><span class="conf-icml">ICML 2026</span><a href="https://arxiv.org/abs/2602.18145" class="bp">paper</a><a href="https://github.com/siyaqi/FrequencyAwareHallucination" class="bc">code</a><span class="pub-title">Detecting Contextual Hallucinations in LLMs with Frequency-Aware Attention</span></div>
<div class="pub"><span class="conf-arxiv">arXiv 2026</span><a href="https://arxiv.org/abs/2602.05289" class="bp">paper</a><span class="pub-title">Towards a Science of Collective AI: LLM-based Multi-Agent Systems Need a Transition from Blind Trial-and-Error to Rigorous Science</span></div>
<div class="pub"><span class="conf-acl">ACL 2025</span><a href="https://arxiv.org/pdf/2406.08979" class="bp">paper</a><a href="https://github.com/OpenBMB/ChatDev" class="bc">code</a><span class="pub-title">Multi-Agent Software Development through Cross-Team Collaboration♣️</span></div>
<div class="pub"><span class="conf-arxiv">arXiv 2024</span><a href="https://arxiv.org/pdf/2405.04219" class="bp">paper</a><a href="https://github.com/OpenBMB/ChatDev" class="bc">code</a><span class="pub-title">Iterative Experience Refinement of Software-Developing Agents</span></div>
<div class="pub"><span class="conf-ccis">CCIS 2018</span><a href="https://www.researchgate.net/publication/332432404_A_Multi-View_Abstractive_Summarization_Model_Jointly_Considering_Semantics_and_Sentiment" class="bp">paper</a><span class="pub-title">A Multi-View Abstractive Summarization Model Jointly Considering Semantics and Sentiment</span></div>
</div>
</div>
</details>

# Social Media

<div class="sm-grid">
<div class="sm-card sm-wechat"><img class="sm-qr" src="/img/social/wechat_qr.png" alt="WeChat QR code: thinkwee的随笔"><div class="sm-text"><div class="sm-meta">WeChat 微信公众号</div><div class="sm-name">thinkwee的随笔</div><div class="sm-desc">Scan the code, or search “thinkwee的随笔” in WeChat.</div></div></div>
<div class="sm-card sm-rednote"><img class="sm-qr" src="/img/social/rednote_qr.png" alt="Rednote QR code: thinkwee"><div class="sm-text"><div class="sm-meta">Rednote 小红书</div><div class="sm-name">thinkwee</div><div class="sm-desc">小红书号: thinkwee. Scan the code to follow.</div></div></div>
</div>

<div class="sm-note">I sometimes sync some articles from this blog on Wechat and Rednote, and I share other interesting things on these platforms.</div>

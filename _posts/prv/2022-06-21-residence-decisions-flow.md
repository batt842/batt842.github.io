---
layout: post
title: "2023 거주지 선택 계획"
date: 2022-06-21 20:00:00 +0200
categories: prv
description: 
comments: true
permalink: /posts/prv/13/residence-decision
published: true
---
{% raw %}
<div class="mermaid">
flowchart TD
    id-beg([June 2022\nUncertainty ])
    id-fin([Late 2023\nOur lives shall go on])
    
    id1-1(Visit Korea alone ASAP for aprx. 1w)
    id1-2(Come back to Berlin)
    id1-3(Visit Korea with Nam in Dec 2022 aprx. 3w)
    if1{{Evaluate my father's status}}
    
    id2-1(Find a caregiver and move him to Yecheon)
    id2-2(Let him stay where he is)
   
    id3-1(Come back to Berlin)
    if2{{Wait for Nam's application result}}
    
    id4-1(Prepare for moving to Korea\nlooking forward to coming back to Germany)
    if3{{Decide where to go for Nam's Doctor's course}}
    
    id5-1(Learn Germany and prepare for EU's PR)
    
    if4{{Decide where to live in Korea\nbased on my father's status}}
    id6-1(Yecheon)
    id6-2(In/near Seoul)
    
    id-beg-->id1-1-->id1-2-->id1-3-->if1
    if1--good-->id2-1-->id3-1
    if1--bad-->id2-2-->id3-1
    id3-1-->if2
    if2--passed-->id4-1
    if2--failed-->if3
    if3--Korea-->id4-1-->if4
    if3--Germany-->id5-1
    if4---->id6-1
    if4---->id6-2
    
    id5-1-->id-fin
    id6-1-->id-fin
    id6-2-->id-fin
</div>

<script src="https://cdn.jsdelivr.net/npm/mermaid/dist/mermaid.min.js"></script>
<script>mermaid.initialize({startOnLoad:true});
</script>
{% endraw %}
---
layout: post
date: 2026-04-06 15:59:00-0400
inline: true
related_posts: false
---

Mr. Mechanic — An educational game. 
<br>
*Currently in active development.*
<br><br>

<img id="hero-image2"
     src="/assets/img/Blog/CompleteVulkanTutorial/TruckScene.png"
     class="scaled-img70 img-fluid">

<style>
#hero-image2 {
  transition: opacity 5s ease;
}
</style>

<script>
document.addEventListener("DOMContentLoaded", function () {
  const images = [
    "assets/img/Blog/CompleteVulkanTutorial/TruckScene.png",
    "assets/img/Blog/CompleteVulkanTutorial/CarScene.png"
  ];

  let index = 0;
  const img = document.getElementById("hero-image2");

  setInterval(() => {
    img.style.opacity = 0;

    setTimeout(() => {
      index = (index + 1) % images.length;
      img.src = images[index];
      img.style.opacity = 1;
    }, 500);
  }, 3000);
});
</script>

<br><br><br>


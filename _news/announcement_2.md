---
layout: post
date: 2026-04-15 15:59:00-0400
inline: true
related_posts: false
---

A real-time minimal demo built using C++ and Vulkan that demonstrates tiled light culling is now released
<br>
<br>
<a href="https://amrmorsy.gumroad.com/l/LightCullingDemo?layout=profile">⇗ Learn more</a>
<br><br>

<img id="hero-image2"
     src="assets/img/Blog/LightCulling/Dark/Thumbnail.png"
     class="scaled-img70 img-fluid">

<style>
#hero-image2 {
  transition: opacity 5s ease;
}
</style>

<script>
document.addEventListener("DOMContentLoaded", function () {
  const images = [
    "assets/img/Blog/LightCulling/Dark/Thumbnail.png"
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

<br>
<br>
<br>
<br>

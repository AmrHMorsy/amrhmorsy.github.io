---
layout: post
date: 2024-12-20 15:59:00-0400
inline: true
related_posts: false
---

Vejaler Engine — A game engine built using the Vulkan API
<br>
*Currently in active development.*
<br><br>

<img id="hero-image"
     src="/assets/img/Blog/CompleteVulkanTutorial/SpaceShipScene.png"
     class="scaled-img70 img-fluid">

<style>
#hero-image {
  transition: opacity 5s ease;
}
</style>

<script>
document.addEventListener("DOMContentLoaded", function () {
  const images = [
    "assets/img/Blog/CompleteVulkanTutorial/CarScene.png",
    "assets/img/Blog/CompleteVulkanTutorial/SpaceShipScene.png",
    "assets/img/Blog/CompleteVulkanTutorial/PlaneScene.png",
    "assets/img/Blog/CompleteVulkanTutorial/CakeScene.png"
  ];

  let index = 0;
  const img = document.getElementById("hero-image");

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

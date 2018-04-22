magic-skill
===========

收录一些网上的奇技淫巧

Table of content
================
  * [Magic Skill](#magic-skill)
    * [javascript](#javascript)
      * [replace-array-splice-in-long-loop](#replace-array-splice-in-long-loop)


javascript
==========

replace-array-splice-in-long-loop
=================================

请尽量避免在 `for` 循环中使用 `array.splice` 方法，有性能问题。用 `Benchmark` 做了个简单的 [基准测试](https://jsperf.com/splice-in-for-loop-benchmark)，同时放上 [J大的测试](https://jsperf.com/inserting-an-array-within-an-array)

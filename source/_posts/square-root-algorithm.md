---
title: Node.js实现平方根算法
tags: ['算法', 'Node']
date: 2016-11-7
---

前几天遇到一个问题: 如何实现对一个数开平方的算法? 于是便打算亲身实现一个sqrt算法。

<!-- more -->

## 代码

```js
function sqrt(num) {
  let seido = 1e-15 // 表示精度

  // 类似二分算法，取中值，
  // params[m1, m2] m1为区间最小值, m2为区间最大值
  // 比如，计算16的平方根，传入参数为[0, 16]，返回结果为[4,8], 表示计算结果的值在[4, 8]这个区间内
  function getValues(m1, m2) {
    let m = (m1 + m2) / 2
    if(num >= 1) {
      //当开方的值大于1
      while(m * m > num) {
        m2 = m
        m = m / 2
      }
      m1 = m
    } else {
      //当开方的值小于1
      while(m * m < num) {
        m1 = m
        m  = m * 2
      }
      m2 = m
    }
    return [m1, m2]
  }

  var [m1, m2] = [0, num]
  while(true) {
    //无限循环不断缩小查找的区间
    [m1, m2] = getValues(m1, m2)
    //直到达到精度后跳出循环
    if(Math.abs(m1 - m2) < seido) {
      break
    }
  }

  // 最后有一个在精度范围内的最小区间, 在此区间内进行累加或累减枚举, 直到最相似的值
  // 累加或累减取决于哪个峰值与目标值的差距较小
  var mid
  if(Math.abs(num - m1) > Math.abs(num - m2)) {
    mid = m1
    while(mid * mid < num) {
      mid = mid + seido
    }
  } else {
    mid = m2
    while(mid * mid > num) {
      mid = mid - seido
    }
  }
  return mid
}
```

虽然是能计算出一部分数的平方根值了，但是问题还是有的。由于使用了精度和无限循环，当精度出现误差值的时候，该方法会陷入死循环，如精度值小于1e16的时候。因此如果需要使用精度计算的时候，最好就不要使用死循环了。
此处仅提供一种算法思路，至于具体如何解决，此时不加讨论。

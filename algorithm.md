### 二叉树层序遍历

### 找出无序数组中的k，要求k大于前方所有的数且小于其后方所有的数

方案是维护当前前方最大值和当前后方最小值，先从后往前遍历一遍，用数组保存当前值是否小于后方所有的值，再从前往后遍历一遍，得到结果

```javascript
const plan = (nums) => {
  const curMinArr = new Array(nums.length);
  let curMin = Infinity;
  let curMax = -Infinity;
  const res = [];
  for (let i = nums.length - 1; i >= 0; i--) {
    if (nums[i] < curMin) {
      curMin = nums[i];
      curMinArr[i] = true;
    }
  }
  for (let i = 0; i < nums.length; i++) {
    if (nums[i] > curMax) {
      curMax = nums[i];
      if (curMinArr[i]) {
        res.push(nums[i]);
      }
    }
  }
  return res;
};

const nums = [2, 3, 1, 8, 9, 20, 12];
console.log(main(nums));

```


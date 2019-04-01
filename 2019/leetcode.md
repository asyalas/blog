### 第一题


利用hash链表，空间复杂度O(n),时间复杂度O(n)
```js
var twoSum = function( nums , target) {
    var map = new Map()
    for(var i = 0 ; i < nums.length ; i++){
      map.set(nums[i],i)
    }
    for(var j = 0 ; j < nums.length ; j++){
      var reduce = target - nums[j]
      if(map.has(reduce) && map.get(reduce) !== j){
        return [j,map.get(reduce)]
      }
    }
    
};
```
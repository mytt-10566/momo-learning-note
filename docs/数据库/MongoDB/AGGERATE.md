```bash
# 分组统计数量

db.users.aggregate([
  {
    $match: {
      addTime: "2024"
    }
  },
  {
    $group: {
      _id: "$pushTime",
      count: { $sum: 1 }
    }
  }
]);
```
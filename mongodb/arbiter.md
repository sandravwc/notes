---
title: mongodb/arbiter
---

- add arbiter to replica set

  ```js
  cfg = rs.conf();
  cfg.members.push({
    _id: 3,
    host: 'mongo-arbiter.example.internal:27018',
    arbiterOnly: true
  });
  rs.reconfig(cfg)
  ```

- flip an existing member to arbiter

  ```js
  cfg = rs.conf();
  cfg.members.forEach((member) => {
    if (member._id === 3) {
      member.arbiterOnly = true;
    }
  });
  rs.reconfig(cfg)
  // arbiter holds no data, votes only. odd member count is the point
  ```

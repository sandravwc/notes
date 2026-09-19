---
title: mongodb
---

- arbiter...
  ```
  cfg.members.forEach((member) => {
    if (member._id === 3) {
      member.arbiterOnly = true;
    }
  });
  
  
  cfg = rs.conf();
  cfg.members.push({
    _id: 3,
    host: 'q-kst-mongo-shard-02-arbiter.zz:27018',
    arbiterOnly: true
  });
  rs.reconfig(cfg)
  ```

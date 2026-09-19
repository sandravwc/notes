---
title: cheat sheet/jq
tags: [cheat sheet]
---

- get json keys
  ```sh
  jq '[paths(scalars)|map(if type == "number" then "[]" else tostring end)|join(".")]|unique|map(.|= gsub("\\.\\["; "[")|.|= "." + .)'
  ```
- select based on test 
  ```sh
  jq '.items[] | select(.metadata.name|test("ingress")) .spec.containers[].image'
  
  # won't work as intended when used like this:
  jq 'select(.items[].metadata.name|test("ingress")) .items[].spec.containers[].image'
  ```

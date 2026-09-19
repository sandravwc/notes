---
title: cheat sheet/awk
---

- pretty print for building sql queries
  ```sh
  awk -F '.' '
      { 
          lines[NR] = $0
      }
      END {
          for (i = 1; i <= NR; i++) {
              split(lines[i], fields, ".")
              printf "%-34s %s_%s", lines[i], fields[1], fields[2]
              if (i < NR) {
                  printf ",\n"
              } else {
                  printf "\n"
              }
          }
      }
  '
  ```
- gsub (search and replace)
  ```sh
  awk '{
    gsub( "*", "'"'*'"'");
    print
  }'
  ```
- gsub with some dots using regex to search and replace
  ```sh
  awk '/Enabling and starting/ {gsub(/\.timer\.\.\./,""); print $NF}' b
  ```
- gsub on line where `^copied` matches, removes single quote, prints last line
- ```sh
  awk -F "/" '/^copied/ { gsub(/\047/, ""); print $NF }' a
  ```

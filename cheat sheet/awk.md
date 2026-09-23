---
title: cheat sheet/awk
---

- pretty print for building sql queries

  ```sh
  awk -F '.' '{ lines[NR] = $0 } END { for (i = 1; i <= NR; i++) { split(lines[i], fields, "."); printf "%-34s %s_%s", lines[i], fields[1], fields[2]; if (i < NR) { printf ",\n" } else { printf "\n" } } }'
  ```

- gsub (search and replace)

  ```sh
  # "*" is a bare regex quantifier and errors; escape it
  awk '{ gsub(/\*/, "'"'*'"'"); print }'
  ```

- gsub escaped dots, print last column

  ```sh
  awk '/Enabling and starting/ {gsub(/\.timer\.\.\./,""); print $NF}' b
  ```

- gsub octal single quote on `^copied` lines, print last column

  ```sh
  awk -F "/" '/^copied/ { gsub(/\047/, ""); print $NF }' a
  ```

# AGENTS.md

Single-file C++ FizzBuzz. No build system, no tests, no CI — compile manually.

## Build & run

```sh
c++ -o fizbuzz fizbuzz.cpp && ./fizbuzz
```

The `fizbuzz` / `*.out` executable is gitignored; do not commit build artifacts.

## Notes

- All logic lives in `fizbuzz.cpp:main`. It builds the output string by appending
  `"Fizz"` (i % 3) then `"Buzz"` (i % 5), so multiples of 15 print `FizzBuzz` by
  design. This is intentional, not a bug — keep the append pattern if editing.

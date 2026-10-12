When you implement you must follow these rules.

- For types that are shared between dpa and host, try to use the same type name from the dpa. That is, use typedef in the host code to match the types. Do not write the right type in the comment.

- You should never make a new struct or a class if not spoken with me. However, you may find a helper class would suit better while implementing. Then, you must ask for me the design.

- You must not add scheduling nor extra parallelism if not asked. However, at the end of the session you may propose such optimizations. There, you must explain with examples.

- Before making a name, see if that conflicts with previous codebase.

- For indirections, you should comment the concurrency model the indirections expect.

- You must ask for me on adding new APIs. That is, you may add class' methods or static functions within a file as you want. However, API functions that can be used from others need my approvals. 

- You should distinguish bench with test. Every bench is run after the test verifies. Thus, no hard checks should exist in the benchmarks. It may assume semantics checked in the test to be true. 

- Simple tests. You must write a simplistic test with only the APIs and class/struct open to the user.

```
void intialize() {
  // initialize here
}

void terminate() {
    // free and others
}

int main() {
  initialize();
  // do main logic. our case it would be io_pool
  terminate();
}
```
Simple test should not use helper functions. They must use from the include headers. The goal of simple tests are making it sure you and me agree on the API presented to the user.

- If not waiting for a channel or the thread actually sleeps, we name it poll.

- YOu should not make APIs dedicated for test purpose. They must be either inside TEST macro or never implemented. Thus, you should not write API by tests but write tests to use those APIs.

- For VCS, this project uses jj. Separate logic change with API change. That is, distinguish in the VCS level if the test files are changed due to the API change.

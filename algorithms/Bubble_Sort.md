# Bubble Sort Algorithm

## Algorithm

Bubble Sort is a simple comparison-based sorting algorithm that utilizes the **Adjacent Swapping Technique**. It relies on the principle of local ordering to achieve global ordering.

### Step-by-Step Process

1. **Start** with an unsorted array of $n$ elements.
2. **Initialize** a boolean variable `swapped` to `true`. This tracks if any elements were moved during a pass.
3. **Enter a loop** that continues as long as `swapped` is `true`.
4. **Set `swapped` to `false`** at the beginning of each pass.
5. **Iterate** through the array from the first element to the second-to-last element (index $i$ from $0$ to $n-2$):
   * Compare the current element $A[i]$ with the next element $A[i+1]$.
   * If $A[i] > A[i+1]$ (they are in the wrong order):
     * **Swap** $A[i]$ and $A[i+1]$.
     * Set `swapped` to `true` to indicate a change was made.
6. **Repeat** the pass. With each full pass, the largest unsorted element "bubbles up" to its correct position at the end of the array.
7. **Terminate** when a full pass completes without any swaps (`swapped` remains `false`).

### Approach Explanation

* **The Swap:** By using a temporary variable (or language-specific swap syntax), we exchange the positions of two adjacent elements. This ensures no data is lost during the overwrite process.
* **The Bubbling Effect:** In each iteration, the algorithm compares neighbors. If the left neighbor is larger, they swap. This causes the largest value in the current unsorted portion to "sink" to the right.
* **Efficiency Check:** The `swapped` flag is a crucial optimization. If the algorithm scans the entire list and finds no swaps were necessary, it concludes the list is already sorted and exits early.

---

## Pseudocode

```text
procedure bubbleSort(A: list of sortable items)
    n := length(A)
    repeat
        swapped := false
        for i := 0 to n - 2 do
            if A[i] > A[i + 1] then
                // The Swapping Technique
                temp := A[i]
                A[i] := A[i + 1]
                A[i + 1] := temp

                swapped := true
            end if
        end for
        // Optimization: reduce n because the last element is sorted
        n := n - 1
    until not swapped
end procedure
```

---

## Flowchart

```text
       +-----------------------+
       |         Start         |
       +-----------+-----------+
                   |
          +--------v---------+
          |  swapped = true  |
          +--------+---------+
                   |
          +--------v---------+ <----------+
          | swapped = false  |           |
          +--------+---------+           |
                   |                     |
          +--------v---------+           |
    +---->|  For i = 0 to n-2|           |
    |     +--------+---------+           |
    |              |                     |
    |     +--------v---------+           |
    |     |  Is A[i] > A[i+1]?|-- No ----+ (Next i)
    |     +--------+---------+           |
    |              | Yes                 |
    |     +--------v---------+           |
    |     | Swap A[i], A[i+1]|           |
    |     |  swapped = true  |           |
    |     +--------+---------+           |
    |              |                     |
    +------- End of Loop?                |
                   | Yes                 |
          +--------v---------+           |
          |  Is swapped true?|-- Yes ----+
          +--------+---------+
                   | No
          +--------v---------+
          |       End        |
          +------------------+
```

---

## Golang Implementation

```go
package main

import (
    "fmt"
)

// BubbleSort implements the adjacent swapping technique
func BubbleSort(arr []int) {
    n := len(arr)
    for {
        swapped := false
        for i := 0; i < n-1; i++ {
            if arr[i] > arr[i+1] {
                // The Swapping Technique
                arr[i], arr[i+1] = arr[i+1], arr[i]
                swapped = true
            }
        }
        // Optimization: The largest element is now at the end
        n--
        // Terminate if no swaps occurred
        if !swapped || n <= 1 {
            break
        }
    }
}

func main() {
    var size int
    fmt.Print("Enter number of elements: ")
    if _, err := fmt.Scan(&size); err != nil || size <= 0 {
        fmt.Println("Invalid input size.")
        return
    }

    nums := make([]int, size)
    fmt.Printf("Enter %d integers separated by spaces: ", size)
    for i := 0; i < size; i++ {
        if _, err := fmt.Scan(&nums[i]); err != nil {
            fmt.Println("Invalid integer input.")
            return
        }
    }

    fmt.Println("Unsorted:", nums)
    BubbleSort(nums)
    fmt.Println("Sorted:  ", nums)
}
```

---

## Python Implementation

```python
def bubble_sort(arr: list[int]) -> None:
    """
    Sorts a list of integers in-place using the Bubble Sort algorithm.
    """
    n = len(arr)

    while True:
        swapped = False
        # Iterate from the first element to the second-to-last
        for i in range(n - 1):
            if arr[i] > arr[i + 1]:
                # The Swapping Technique (Pythonic way)
                arr[i], arr[i + 1] = arr[i + 1], arr[i]
                swapped = True

        # Optimization: reduce n because the last element is sorted
        n -= 1

        # Terminate if a full pass completes without any swaps
        if not swapped or n <= 1:
            break

def main():
    try:
        user_input = input("Enter integers separated by spaces: ")
        # Safe parsing of input string to list of integers
        nums = [int(item) for item in user_input.split()]

        if not nums:
            print("The list is empty.")
            return

        print(f"Unsorted: {nums}")
        bubble_sort(nums)
        print(f"Sorted:   {nums}")

    except ValueError:
        print("Error: Please enter valid integers only.")

if __name__ == "__main__":
    main()
```

---

## Rust Implementation

```rust
use std::io::{self, Write};

/// Performs bubble sort on a mutable slice of integers
fn bubble_sort(arr: &mut [i32]) {
    let mut n = arr.len();
    if n == 0 {
        return;
    }

    loop {
        let mut swapped = false;
        for i in 0..n - 1 {
            if arr[i] > arr[i + 1] {
                // Idiomatic Rust swapping
                arr.swap(i, i + 1);
                swapped = true;
            }
        }

        // Optimization: reduce n because the last element is sorted
        n -= 1;

        // Terminate when no swaps occur or range is exhausted
        if !swapped || n <= 1 {
            break;
        }
    }
}

fn main() {
    print!("Enter integers separated by spaces: ");
    io::stdout().flush().unwrap();

    let mut input = String::new();
    io::stdin().read_line(&mut input).expect("Failed to read line");

    // Safe input parsing into a Vector
    let mut nums: Vec<i32> = input
        .split_whitespace()
        .filter_map(|s| s.parse::<i32>().ok())
        .collect();

    if nums.is_empty() {
        println!("No valid integers provided.");
        return;
    }

    println!("Unsorted: {:?}", nums);
    bubble_sort(&mut nums);
    println!("Sorted:   {:?}", nums);
}
```

---

## Test Cases

### 1. Example Scenarios

| Scenario              | Input                          | Expected Output                |
|:--------------------- |:------------------------------ |:------------------------------ |
| **Standard Unsorted** | `[64, 34, 25, 12, 22, 11, 90]` | `[11, 12, 22, 25, 34, 64, 90]` |
| **Already Sorted**    | `[1, 2, 3, 4, 5]`              | `[1, 2, 3, 4, 5]`              |
| **Reverse Sorted**    | `[5, 4, 3, 2, 1]`              | `[1, 2, 3, 4, 5]`              |
| **Duplicates**        | `[3, 1, 4, 1, 5, 9, 2, 6, 5]`  | `[1, 1, 2, 3, 4, 5, 5, 6, 9]`  |
| **Negative Numbers**  | `[-5, 2, -10, 0, 7]`           | `[-10, -5, 0, 2, 7]`           |

### 2. Python Test Harness

```python
import unittest

class TestBubbleSort(unittest.TestCase):
    def test_standard_list(self):
        data = [64, 34, 25, 12, 22, 11, 90]
        bubble_sort(data)
        self.assertEqual(data, [11, 12, 22, 25, 34, 64, 90])

    def test_already_sorted(self):
        data = [1, 2, 3, 4, 5]
        bubble_sort(data)
        self.assertEqual(data, [1, 2, 3, 4, 5])

    def test_reverse_sorted(self):
        data = [5, 4, 3, 2, 1]
        bubble_sort(data)
        self.assertEqual(data, [1, 2, 3, 4, 5])

    def test_duplicates(self):
        data = [3, 1, 4, 1, 5, 2]
        bubble_sort(data)
        self.assertEqual(data, [1, 1, 2, 3, 4, 5])

    def test_negatives(self):
        data = [-5, 2, -10, 0, 7]
        bubble_sort(data)
        self.assertEqual(data, [-10, -5, 0, 2, 7])

    def test_empty_list(self):
        data = []
        bubble_sort(data)
        self.assertEqual(data, [])

    def test_single_element(self):
        data = [42]
        bubble_sort(data)
        self.assertEqual(data, [42])

if __name__ == "__main__":
    unittest.main()
```

---

## Complexity Analysis

* **Time Complexity:**
  * **Worst Case:** $O(n^2)$ — Occurs when the array is in reverse order.
  * **Average Case:** $O(n^2)$ — Occurs with random distribution.
  * **Best Case:** $O(n)$ — Occurs when the array is already sorted (enabled by the `swapped` flag optimization).
* **Space Complexity:** $O(1)$ — It is an **in-place** sorting algorithm, requiring only a constant amount of additional memory regardless of input size.

---
description: https://go101.org/quizzes/101.html
---

# Quizzes

{% code overflow="wrap" lineNumbers="true" expandable="true" %}
```go
package main

import "fmt"

func main() {
	a := [...]int{0, 1, 2, 3} // len is 4
	x := a[:1] // [0] correct
	fmt.Println(x)
	 
	y := a[2:] // [2,3] correct 
	fmt.Println(y)
	
	x = append(x, y...) // [0,2,3] correct
	fmt.Println(x)
	
	x = append(x, y...) // [0,2,3,2,3] wrong
	fmt.Println(a, x)   // [0,1,2,3] [0,2,3,2,3] wrong
}

/*
[0, 1, 2, 3]
 ^     ^__^
 x       y

[0, 2, 3, 3]
 ^_____^__^
    x    y

[0, 2, 3,| 3, 3]
 ^     ^__^
 x       y
*/
```
{% endcode %}

O/P:

{% code overflow="wrap" lineNumbers="true" expandable="true" %}
```









[0]
[2 3]
[0 2 3]
[0 2 3 3] [0 2 3 3 3]
```
{% endcode %}



{% code overflow="wrap" lineNumbers="true" expandable="true" %}
```go
package main

import "fmt"

func modify(s []int) {
	s = append(s, 100)
	s[0] = 999
}

func main() {
	a := []int{1, 2, 3, 4, 5}
	b := a[1:3] // Len: 2, Cap: 4 -> [2, 3]

	modify(b)

	fmt.Println("a:", a)
	fmt.Println("b:", b)
}
```
{% endcode %}

{% code overflow="wrap" lineNumbers="true" expandable="true" %}
```














```
{% endcode %}

---
title: "Callback Function in Golang"
date: 2018-11-03T00:00:00+05:30
draft: false
author: "Oshank Kumar"
tags:
  - Golang
  - Callback
image: /images/blogs/go_generic.png
description: ""
toc: true
---

---

# Introduction

Passing a function as an argument is called callback in golang. Here, we can pass a function as an argument.

In this example, we are checking whether a given sequence is in AP(arithmetic progression) or in GP(geometric progression) by using callback.

First, we are reading the length of the seq from STDIN then we are reading the entire sequence.

```go
package main

import (
   "fmt"
   "io"
   "os"
)

var ConditionMap = map[string]func(int, int) int{
   "AP": func(i1 int, i2 int) int {
      return i2 - i1
   },
   "GP": func(i1 int, i2 int) int {
      return i2 / i1
   },
}

func main() {
   var seqLen int
   fmt.Scan(&seqLen)

   s := make(Sequence, seqLen)
   s.ReadInput(os.Stdin)

   switch {
   case s.ExamineProgression(ConditionMap["AP"]):
      fmt.Println("AP")
   case s.ExamineProgression(ConditionMap["GP"]):
      fmt.Println("GP")
   default:
      fmt.Println("RANDOM")
   }
}

type Sequence []int

func (s Sequence) ReadInput(r io.Reader) {
   for j := 0; j < len(s); j++ {
      fmt.Fscan(r, &s[j])
   }
}

func (s Sequence) ExamineProgression(condition func(a, b int) int) bool {
   if len(s) < 3 {
      return false
   }
   d := condition(s[1], s[0])
   for i := 2; i < len(s)-1; i++ {
      if condition(s[i+1], s[i]) != d {
         return false
      }
   }
   return true
}
```

Here In ExamineProgression method, we are passing a function as a parameter. It is very useful in reusing the code.

Other wise we have to write separate methods to check the sequence whether it is an AP or GP.

